# ansible-ollama

Déploiement d'Ollama en conteneur Podman rootless (quadlet systemd) sur une
Bazzite, piloté depuis le Mac via SSH sur le tailnet. Sert l'API Ollama brute
sur le port 11434, sans OpenWebUI ni reverse proxy (Tailscale chiffre déjà le
trafic). Génère aussi la config OpenCode côté Mac pour pointer dessus.

## Arborescence

```
ansible.cfg
inventory.yml
vars.yml                         source de vérité (GPU, port, modèles, host)
site.yml                         2 plays : Ollama (Bazzite) + OpenCode (Mac)
templates/
  ollama.container.j2            quadlet Podman
  opencode.jsonc.j2             config OpenCode
```

## Prérequis

Côté Mac :
- Ansible installé (`brew install ansible`)
- Tailscale actif, la Bazzite visible dans `tailscale status`
- SSH qui passe : `ssh marcfusch@bazzite-8086k.tail72eeaa.ts.net`
  (idéalement Tailscale SSH activé sur la Bazzite : `sudo tailscale up --ssh`)

Côté Bazzite (GPU NVIDIA) :
- Le NVIDIA Container Toolkit doit être configuré avec une spec CDI, sinon le
  conteneur ne verra pas la carte. Sur Bazzite :

  ```bash
  ujust setup-nvidia-container-toolkit   # si la recette existe sur ton image
  # sinon, générer la spec CDI manuellement :
  sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
  nvidia-ctk cdi list                    # doit lister nvidia.com/gpu=all
  ```

  C'est ce `nvidia.com/gpu=all` qui est référencé par `AddDevice` dans le
  quadlet.

## Configuration

Tout est dans `vars.yml` : type de GPU (`ollama_gpu`), image, port, longueur de
contexte, liste des modèles à tirer, adresse tailnet de la Bazzite et modèle
OpenCode par défaut. La config OpenCode (baseURL + liste de modèles) en dérive,
donc un modèle ajouté à `ollama_models` apparaît des deux côtés. Pour basculer
sur AMD, mettre `ollama_gpu: amd` et décommenter l'image `:rocm`.

## Déploiement

```bash
ansible bazzite-gpu -m ping          # vérifier la connexion
ansible-playbook site.yml            # Ollama sur la Bazzite + config OpenCode
```

Pour ne jouer qu'une moitié : `--limit bazzite-gpu` (Ollama seul) ou
`--limit localhost` (OpenCode seul).

## Vérifications post-déploiement

Sur la Bazzite :

```bash
systemctl --user status ollama
curl http://localhost:11434/api/tags
# après un premier prompt, vérifier que le GPU est bien utilisé :
ollama ps                            # colonne PROCESSOR doit montrer du GPU
```

Depuis OpenCode sur le Mac, pointer vers
`http://bazzite-8086k.tail72eeaa.ts.net:11434/v1`.
