<template>
  <div>
    <link rel="stylesheet" href="https://releases.transloadit.com/uppy/v3.13.0/uppy.min.css">
    <div ref="uppyRoot"></div>
  </div>
</template>

<script>
import Uppy from "@uppy/core";
import Dashboard from "@uppy/dashboard";
import AwsS3 from "@uppy/aws-s3";

export default {
  props: {
    presignEndpoint: { type: String, required: true },
    supabaseAnonKey: { type: String, required: true },
    bearerToken: { type: String, default: "" },
    keyPrefix: { type: String, default: "uploads" },
    expires: { type: Number, default: 3600 },
  },
  data() {
    return { uppy: null };
  },
  mounted() {
    const sanitizeName = (name) => String(name || "file").replace(/[^\w.\-]+/g, "_");

    const uppy = new Uppy({ autoProceed: false });

    uppy.use(Dashboard, {
      inline: true,
      target: this.$refs.uppyRoot,
      proudlyDisplayPoweredByUppy: false,
    });

    uppy.use(AwsS3, {
      getUploadParameters: async (file) => {
        const safe = sanitizeName(file.name);
        const key = `${this.keyPrefix}/${Date.now()}_${safe}`;

        const headers = {
          "Content-Type": "application/json",
          apikey: this.supabaseAnonKey,
        };
        if (this.bearerToken) headers.Authorization = `Bearer ${this.bearerToken}`;

        const res = await fetch(this.presignEndpoint, {
          method: "POST",
          headers,
          body: JSON.stringify({ key, op: "put", expires: this.expires }),
        });

        if (!res.ok) {
          const txt = await res.text().catch(() => "");
          throw new Error(`presign failed: ${res.status} ${txt}`);
        }

        const data = await res.json();
        if (!data || !data.url) throw new Error("presign response missing url");

        return {
          method: "PUT",
          url: data.url,
          fields: {},
          headers: { "Content-Type": file.type || "application/octet-stream" },
        };
      },
    });

    uppy.on("upload-success", (file, response) => {
      this.$emit("upload-success", { name: file.name, size: file.size, response });
    });

    uppy.on("upload-error", (file, error) => {
      this.$emit("upload-error", { name: file && file.name, error: String((error && error.message) || error) });
    });

    this.uppy = uppy;
  },
  beforeDestroy() {
    try { this.uppy && this.uppy.close && this.uppy.close(); } catch (_) {}
  },
};
</script>
