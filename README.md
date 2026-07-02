# ansible-ollama

Déploiement d'Ollama en conteneur Podman rootless (quadlet systemd) sur une
Bazzite, piloté depuis le Mac via SSH sur le tailnet. Sert l'API Ollama brute
sur le port 11434, sans OpenWebUI ni reverse proxy (Tailscale chiffre déjà le
trafic).

## Arborescence

```
ansible.cfg
inventory.yml
playbook.yml
vars.yml
roles/
  ollama/
    tasks/main.yml
    handlers/main.yml
    templates/ollama.container.j2
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
contexte et liste des modèles à tirer. Pour basculer sur AMD, mettre
`ollama_gpu: amd` et décommenter l'image `:rocm`.

## Déploiement

```bash
ansible bazzite-gpu -m ping          # vérifier la connexion
ansible-playbook playbook.yml        # déployer
```

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
