# -alerte-rdv.
#!/usr/bin/env python3
"""Vérification unique des créneaux (pour GitHub Actions).

Variables d'environnement (à définir dans les "secrets" GitHub) :
  URL_RDV     adresse de la page de rendez-vous
  NTFY_TOPIC  nom secret de ton sujet ntfy
"""

import os
import re
import sys
import urllib.request
from html import unescape

URL_RDV = os.environ.get("URL_RDV", "")
NTFY_TOPIC = os.environ.get("NTFY_TOPIC", "")

# À adapter selon le texte affiché quand il n'y a plus de créneau.
PHRASES_COMPLET = [
    "aucun créneau",
    "aucune disponibilité",
    "aucun rendez-vous disponible",
    "plus de créneaux",
    "plus aucun créneau",
    "pas de créneau",
    "aucune plage horaire",
    "n'est disponible",
]


def notifier(titre, message):
    req = urllib.request.Request(
        f"https://ntfy.sh/{NTFY_TOPIC}",
        data=message.encode("utf-8"),
        headers={"Title": titre.encode("utf-8"), "Priority": "urgent",
                 "Tags": "rotating_light", "Click": URL_RDV},
        method="POST",
    )
    urllib.request.urlopen(req, timeout=20).read()


def main():
    if not URL_RDV or not NTFY_TOPIC:
        sys.exit("URL_RDV et NTFY_TOPIC doivent être définis.")

    req = urllib.request.Request(URL_RDV, headers={"User-Agent": "Mozilla/5.0"})
    try:
        with urllib.request.urlopen(req, timeout=30) as r:
            html = r.read().decode(r.headers.get_content_charset() or "utf-8",
                                   errors="replace")
    except Exception as e:
        print(f"Erreur de lecture : {e}")
        return  # on réessaiera au prochain passage

    html = re.sub(r"(?is)<(script|style).*?</\1>", " ", html)
    texte = unescape(re.sub(r"(?s)<[^>]+>", " ", html)).lower().replace("’", "'")
    texte = re.sub(r"\s+", " ", texte)

    if any(p in texte for p in PHRASES_COMPLET):
        print("Complet pour le moment.")
    else:
        print("Créneaux possibles ! Envoi de l'alerte.")
        notifier("Créneau dispo : commission médicale",
                 "La page n'affiche plus « complet ». Va vite réserver :\n" + URL_RDV)


if __name__ == "__main__":
    main()
