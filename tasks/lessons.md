# Leçons Apprises

## Configuration Git avec les Dépôts Imbriqués (Sous-modules)

**Problème :** L'ajout du dossier `.agent/skills` avec `git add` créait un *"embedded repository"* (dépôt imbriqué) car il contenait son propre dossier `.git`. Cela empêche un suivi propre et bloque souvent les commits ou le travail collaboratif.
**Solution :** Utiliser `git rm --cached -f <dossier>` pour le retirer de l'index, puis déclarer formellement le sous-module avec `git submodule add <url> <dossier>`. Git crée alors un fichier `.gitmodules` qui assure la portabilité correcte.
