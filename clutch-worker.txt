// Clutch Alert worker: runs every minute on Cloudflare, checks NBA games,
// and posts to a Discord channel (your phone gets a push) when a game is close in the final minutes.
//
// Setup values (Worker > Settings):
//   Secret    DISCORD_WEBHOOK  required, the Discord webhook URL
//   Variable  MARGIN      optional, points apart that counts as close (default 5)
//   Variable  WINDOW_SEC  optional, seconds left in the 4th quarter or OT (default 180)
//   KV binding  SEEN      required, remembers which games already alerted

const ESPN = "https://site.api.espn.com/apis/site/v2/sports/basketball/nba/scoreboard";

function parse(ev) {
  const comp = (ev.competitions && ev.competitions[0]) || {};
  const st = ev.status || comp.status || {};
  const type = st.type || {};
  const g = {
    id: ev.id,
    state: type.state || "pre",
    period: st.period || 0,
    clock: Number(st.clock) || 0,
    display: st.displayClock || "",
    home: { name: "TBD", score: 0 },
    away: { name: "TBD", score: 0 },
  };
  for (const c of comp.competitors || []) {
    const side = c.homeAway === "home" ? g.home : g.away;
    side.name = (c.team && (c.team.abbreviation || c.team.shortDisplayName)) || "TBD";
    side.score = Number(c.score) || 0;
  }
  return g;
}

function isClutch(g, margin, windowSec) {
  return (
    g.state === "in" &&
    g.period >= 4 &&
    g.clock <= windowSec &&
    Math.abs(g.home.score - g.away.score) <= margin
  );
}

function periodLabel(p) {
  return p <= 4 ? "Q" + p : p === 5 ? "OT" : p - 4 + "OT";
}

async function push(env, title, body) {
  const res = await fetch(env.DISCORD_WEBHOOK, {
    method: "POST",
    headers: { "Content-Type": "application/json", "User-Agent": "clutch-alert-worker" },
    body: JSON.stringify({ content: "**" + title + "**\n" + body }),
  });
  if (!res.ok) throw new Error("Discord returned " + res.status);
}

async function check(env, { test = false } = {}) {
  if (!env.DISCORD_WEBHOOK) throw new Error("DISCORD_WEBHOOK is not set");
  if (test) {
    await push(env, "Clutch Alert test", "If you can read this, alerts work.");
    return { test: "sent" };
  }
  const margin = Number(env.MARGIN) || 5;
  const windowSec = Number(env.WINDOW_SEC) || 180;

  const res = await fetch(ESPN, { headers: { accept: "application/json" } });
  if (!res.ok) throw new Error("ESPN returned " + res.status);
  const data = await res.json();
  const games = (data.events || []).map(parse);

  const sent = [];
  for (const g of games) {
    if (!isClutch(g, margin, windowSec)) continue;
    const key = "alert:" + g.id + ":" + g.period;
    if (await env.SEEN.get(key)) continue;
    const diff = Math.abs(g.home.score - g.away.score);
    const title = g.away.name + " " + g.away.score + ", " + g.home.name + " " + g.home.score;
    const body =
      (diff === 0 ? "Tied" : "Within " + diff) + " with " + g.display + " left in " + periodLabel(g.period);
    await push(env, title, body);
    await env.SEEN.put(key, "1", { expirationTtl: 6 * 60 * 60 });
    sent.push(title);
  }
  return { checked: games.length, live: games.filter((g) => g.state === "in").length, sent };
}

export default {
  async scheduled(event, env, ctx) {
    ctx.waitUntil(check(env));
  },
  // Open the worker's URL to run a check by hand. Add ?test=1 to send a test push.
  async fetch(request, env) {
    try {
      const test = new URL(request.url).searchParams.get("test") === "1";
      return Response.json(await check(env, { test }));
    } catch (e) {
      return Response.json({ error: String(e.message || e) }, { status: 500 });
    }
  },
};
