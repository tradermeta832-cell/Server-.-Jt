
import express from "express";
import helmet from "helmet";
import rateLimit from "express-rate-limit";

const app = express();
const PORT = process.env.PORT || 3000;
const API_KEY = process.env.GAMEDATA365_API_KEY;
const API_BASE = "https://api.gamedata365.com";

app.use(helmet());
app.use(express.static("public"));

app.use("/api", rateLimit({
  windowMs: 60 * 1000,
  limit: 60
}));

app.get("/api/health", (req, res) => {
  res.json({
    status: "ok",
    apiKeyConfigured: Boolean(API_KEY)
  });
});

app.get("/api/crash/history", async (req, res) => {
  if (!API_KEY) {
    return res.status(503).json({
      error: "Clé API non configurée sur le serveur."
    });
  }

  const game = String(req.query.game || "");
  const limit = Math.min(
    Math.max(Number(req.query.limit) || 10, 1),
    100
  );

  if (!game || !/^[a-zA-Z0-9_-]+$/.test(game)) {
    return res.status(400).json({
      error: "Identifiant du jeu invalide."
    });
  }

  try {
    const url = new URL("/v1/crash/history", API_BASE);
    url.searchParams.set("game", game);
    url.searchParams.set("limit", String(limit));

    const response = await fetch(url, {
      headers: { "X-API-Key": API_KEY }
    });

    const data = await response.json();

    if (!response.ok) {
      return res.status(response.status).json({
        error: "Erreur de récupération des données."
      });
    }

    res.json(data);
  } catch (error) {
    res.status(502).json({
      error: "Impossible de contacter GameData365."
    });
  }
});

app.get("/api/crash/live", async (req, res) => {
  if (!API_KEY) {
    return res.status(503).json({
      error: "Clé API non configurée."
    });
  }

  try {
    const response = await fetch(
      `${API_BASE}/v1/crash/live`,
      { headers: { "X-API-Key": API_KEY } }
    );

    const data = await response.json();

    if (!response.ok) {
      return res.status(response.status).json({
        error: "Flux en direct indisponible."
      });
    }

    res.json(data);
  } catch {
    res.status(502).json({
      error: "Connexion à GameData365 impossible."
    });
  }
});

app.listen(PORT, () => {
  console.log(`JETX POWER 5X lancé sur le port ${PORT}`);
});
