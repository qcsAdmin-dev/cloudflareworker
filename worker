export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // Health check
    if (request.method === "GET" && url.pathname === "/health") {
      return new Response("ok");
    }

    // Upload endpoint
    if (request.method === "POST" && url.pathname === "/upload") {
      // Simple bearer auth
      const auth = request.headers.get("authorization") || "";
      const token = auth.startsWith("Bearer ") ? auth.slice(7) : "";
      if (!token || token !== env.UPLOAD_TOKEN) {
        return json({ error: "unauthorized" }, 401);
      }

      const productId = (request.headers.get("x-product-id") || "").trim();
      const viewIndex = (request.headers.get("x-view-index") || "").trim(); // "01".."06"
      const ext = (request.headers.get("x-ext") || "png").trim().toLowerCase();

      if (!productId) return json({ error: "missing x-product-id" }, 400);
      if (!viewIndex) return json({ error: "missing x-view-index" }, 400);
      if (!/^\d{2}$/.test(viewIndex)) return json({ error: "x-view-index must be 2 digits like 01" }, 400);
      if (!/^(png|jpg|jpeg|webp)$/.test(ext)) return json({ error: "unsupported ext" }, 400);

      // Read body as bytes
      const bytes = await request.arrayBuffer();
      if (!bytes || bytes.byteLength === 0) return json({ error: "empty body" }, 400);

      // Deterministic object key
      const safeProduct = safeSegment(productId);
      const key = `products/${safeProduct}/view_${viewIndex}.${ext}`;

      // Store in R2
      const contentType = ext === "jpg" || ext === "jpeg" ? "image/jpeg"
                        : ext === "webp" ? "image/webp"
                        : "image/png";

      await env.BUCKET.put(key, bytes, {
        httpMetadata: {
          contentType,
          cacheControl: "public, max-age=31536000, immutable",
        },
      });

      // Public URL assumes you attached the bucket to a custom domain:
      const base = (env.PUBLIC_BASE_URL || "").replace(/\/+$/, "");
      const publicUrl = `${base}/${key}`;

      return json({ ok: true, key, publicUrl }, 200);
    }

    return json({ error: "not_found" }, 404);
  },
};

function json(obj, status = 200) {
  return new Response(JSON.stringify(obj), {
    status,
    headers: { "content-type": "application/json; charset=utf-8" },
  });
}

function safeSegment(s) {
  return String(s).trim().replace(/[^\w\-]+/g, "_").slice(0, 120);
}
