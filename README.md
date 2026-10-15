const cors = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "Content-Type",
};
const json = (o, s = 200) =>
  new Response(JSON.stringify(o), {
    status: s,
    headers: { ...cors, "Content-Type": "application/json" },
  });

export async function onRequest(context) {
  const { request, env } = context;
  if (request.method === "OPTIONS") {
    return new Response(null, { headers: cors });
  }
  const url = new URL(request.url);
  const ref = url.searchParams.get("reference");
  if (!ref) return json({ ok: false }, 400);

  if (env.PAID && (await env.PAID.get(ref))) return json({ ok: true });

  const r = await fetch(
    "https://api.paystack.co/transaction/verify/" + encodeURIComponent(ref),
    { headers: { Authorization: "Bearer " + env.PAYSTACK_SECRET } }
  );
  const d = await r.json();
  if (
    d.status &&
    d.data &&
    d.data.status === "success" &&
    d.data.amount >= Number(env.PRICE_KOBO)
  ) {
    if (env.PAID) await env.PAID.put(ref, "1");
    return json({ ok: true });
  }
  return json({ ok: false });
}
