<template>
  <div>
    <div ref="uppyRoot"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";

import Uppy from "@uppy/core";
import Dashboard from "@uppy/dashboard";
import AwsS3 from "@uppy/aws-s3";

import "@uppy/core/dist/style.css";
import "@uppy/dashboard/dist/style.css";

const props = defineProps({
  presignEndpoint: { type: String, required: true },
  supabaseAnonKey: { type: String, required: true },
  bearerToken: { type: String, default: "" },
  keyPrefix: { type: String, default: "uploads" },
  expires: { type: Number, default: 3600 },
});

const emit = defineEmits(["upload-success", "upload-error"]);

const uppyRoot = ref(null);
let uppy;

function sanitizeName(name) {
  return String(name || "file").replace(/[^\w.\-]+/g, "_");
}

onMounted(() => {
  uppy = new Uppy({ autoProceed: false });

  uppy.use(Dashboard, {
    inline: true,
    target: uppyRoot.value,
    proudlyDisplayPoweredByUppy: false,
  });

  uppy.use(AwsS3, {
    async getUploadParameters(file) {
      const safe = sanitizeName(file.name);
      const key = `${props.keyPrefix}/${Date.now()}_${safe}`;

      const headers = {
        "Content-Type": "application/json",
        apikey: props.supabaseAnonKey,
      };
      if (props.bearerToken) headers["Authorization"] = `Bearer ${props.bearerToken}`;

      const res = await fetch(props.presignEndpoint, {
        method: "POST",
        headers,
        body: JSON.stringify({ key, op: "put", expires: props.expires }),
      });

      if (!res.ok) {
        const txt = await res.text().catch(() => "");
        throw new Error(`presign failed: ${res.status} ${txt}`);
      }

      const data = await res.json();
      if (!data?.url) throw new Error("presign response missing url");

      return {
        method: "PUT",
        url: data.url,
        headers: { "Content-Type": file.type || "application/octet-stream" },
      };
    },
  });

  uppy.on("upload-success", (file, response) => {
    emit("upload-success", { name: file.name, size: file.size, response });
  });

  uppy.on("upload-error", (file, error) => {
    emit("upload-error", { name: file?.name, error: String(error?.message || error) });
  });
});

onBeforeUnmount(() => {
  uppy?.close?.();
});
</script>
