---
description: Cherche un sujet dans la mémoire évolutive (index plein-texte de Vesta), pour le repêchage au rituel de session
argument-hint: [2-3 mots-clés, OR entre synonymes, jamais une question]
---

Tu cherches un sujet dans la mémoire évolutive de l'entreprise (le dépôt `vesta`), via son index plein-texte. Ça sert le rituel de repêchage : avant de bâtir sur un sujet, on retrouve les fils déjà au carnet, parce que le modèle est amnésique et qu'un `grep` rate un fil quand on ne devine pas le bon mot. L'index classe par pertinence (plein-texte Postgres, config française), bien mieux que la recherche de code GitHub, et il fonctionne même quand la session n'a pas le dépôt `vesta` cloné (le cas fréquent en session infonuagique scopée sur un dépôt d'outil).

Le sujet à chercher est dans `$ARGUMENTS`. S'il est vide, demande quoi chercher en une ligne, puis arrête.

## Comment formuler la requête

L'index exige TOUS les mots de la requête dans un même passage (un ET implicite, `websearch_to_tsquery`). Une question ou une phrase complète (« qu'est-ce qu'on a décidé pour la tarification du produit X ») ne matche presque jamais un passage entier : elle rend une liste vide qui ressemble à un index injoignable, alors que le même sujet en deux ou trois mots-clés (« tarification produit X ») rend les bons fils.

- Formule `$ARGUMENTS` en **deux ou trois mots-clés**, jamais une question ni une phrase complète.
- Mets **`OR`** (majuscules) entre des synonymes ou des variantes du même sujet plutôt que de tout mettre dans une seule requête ET (« Danny Tremblay OR client Tremblay »).
- Le sujet a plusieurs angles ? Lance **plusieurs requêtes courtes** l'une après l'autre plutôt qu'une seule requête longue qui les additionne toutes.
- Depuis 2026-09, `/api/chercher` retente lui-même une fois en OU quand la requête ET rend zéro résultat (voir le champ `mode` plus bas) : un filet, pas une raison de revenir à la phrase complète.

## Comment chercher

L'index vit dans `vesta-app`, exposé par l'endpoint `POST https://vesta.gendronfils.ca/api/chercher`. Deux secrets, jamais affichés au chat ni écrits dans une réponse :

- le jeton de lecture dédié `VESTA_CHERCHER_TOKEN` (il n'ouvre que cette route, jamais les crons) ;
- le contournement de protection Vercel `VERCEL_AUTOMATION_BYPASS_SECRET`, requis tant que la protection de déploiement est active sur la prod.

Résolution des secrets, dans cet ordre :

1. **Variables d'environnement** (le canal des sessions infonuagiques : posées dans la config de l'environnement Claude Code, valeurs maîtres dans les variables d'environnement du projet Vercel) : `VESTA_CHERCHER_TOKEN` et `VERCEL_AUTOMATION_BYPASS_SECRET`.
2. **Repli local** (machine de Philippe) : les fichiers `.secrets/vesta-chercher-token.txt` et `.secrets/vesta-vercel-bypass.txt` du dossier Gendron & Fils.

Lis-les depuis bash et utilise-les directement dans la commande `curl`, sans les faire transiter par le chat :

```bash
JETON="${VESTA_CHERCHER_TOKEN:-$(cat ".secrets/vesta-chercher-token.txt" 2>/dev/null)}"
BYPASS="${VERCEL_AUTOMATION_BYPASS_SECRET:-$(cat ".secrets/vesta-vercel-bypass.txt" 2>/dev/null)}"
curl -sS -X POST "https://vesta.gendronfils.ca/api/chercher" \
  -H "Authorization: Bearer ${JETON}" \
  -H "x-vercel-protection-bypass: ${BYPASS}" \
  -H "Content-Type: application/json" \
  --data "$(jq -nc --arg q "$ARGUMENTS" '{question:$q}')"
```

La réponse est `{ "resultats": [ { "chemin": "...", "extrait": "..." }, ... ], "mode": "et" | "ou", "requete_effective": "..." }`, les résultats déjà classés du plus pertinent au moins.

`mode` dit si la requête ET d'origine a suffi (`"et"`) ou si l'index a dû élargir en OU parce que le ET rendait zéro résultat (`"ou"`, un `requete_effective` différent de ta requête d'origine). Un `mode: "ou"` est un résultat **dégradé** : dis-le à Philippe si tu t'en sers pour bâtir une réponse (« l'index a élargi la requête, il n'a rien trouvé pour les mots exacts »), et reformule en deux ou trois mots-clés plus ciblés la prochaine fois plutôt que de t'y fier par défaut.

## Quoi en faire

1. Présente les résultats : pour chacun, le chemin et l'extrait surligné, du plus pertinent au moins. Quelques-uns, pas la liste brute complète.
2. Ouvre les fichiers les plus pertinents pour bâtir ta réponse sur le vrai contenu, pas seulement l'extrait : depuis le clone local de la mémoire s'il est dans la session, sinon par l'API GitHub (`Gendron-Fils/vesta`).
3. **Sur le poste d'Atelier, si aucun extrait ne répond à la question** (ou que l'index ne rend rien) et que la commande `chercher-local` existe, lance-la avec la question complète et tes mots-clés : `chercher-local "<la question en phrase complète>" --mots "<tes mots-clés>"`. Elle cherche dans le clone local de la mémoire, sans réseau ni jeton, et un trieur local reclasse les passages selon la question. Pour chaque résultat, lis environ 60 lignes à partir de la ligne donnée (`chemin:ligne`) plutôt que le fichier entier. Mesuré le 2026-09-25 sur 24 questions dont on connaissait la réponse (poste-de-quart, `journal/2026-09-25-f9-chercher.md`) : l'API seule met le bon fichier dans ses 5 premiers résultats 16 fois. Avec `chercher-local` en second recours, c'est 22 fois, pour environ douze fois moins de jetons.
4. Si l'index ne rend rien ou que l'endpoint est injoignable (secrets absents, réseau) : dis-le franchement et dégrade proprement. Si le dépôt `vesta` est cloné dans la session, retombe sur une recherche locale (`rg -i`) en le signalant ; sinon, dis que la recherche de la mémoire n'est pas disponible dans cette session et ce qui manque (la variable d'environnement). Ne fais jamais semblant d'avoir cherché l'index si tu ne l'as pas atteint.

Note : l'index se remplit et se rafraîchit tout seul (cron quotidien `/api/cron/reindex-memoire` à 7 h de l'Est). Les résultats reflètent la mémoire de la dernière réindexation, pas les commits de la dernière heure.
