# Gestión de credenciales en CaC: cómo evitar secretos en plano en Git

**Destinatario:** Equipo AAP — Mapfre  
**Autor:** Mario Castro Ginés — TAM, Red Hat  
**Fecha:** Septiembre 2026  
**Validado en laboratorio con AAP 2.6 (controller 4.7.16)**

---

## El problema

Cuando se gestionan credenciales de AAP como código (CaC), las definiciones de credenciales incluyen valores sensibles: contraseñas, claves SSH, tokens de API. Estos valores **no pueden ir en texto plano en el repositorio Git**.

---

## Opciones disponibles

### Opción 1 — Ansible Vault (validada en laboratorio)

Cifrar los valores sensibles con `ansible-vault` dentro del propio repositorio. El pipeline descifra con la vault password almacenada como secreto del CI/CD (GitHub Secret, GitLab Variable, etc.).

**Cómo funciona:**

```
Repositorio Git
├── credentials.yml          ← definiciones con {{ vault_* }} (legible, sin secretos)
├── secrets.yml              ← valores cifrados con ansible-vault (ilegible)
├── aplicar_credenciales.yml ← playbook
└── .gitignore               ← excluye .vault_password
```

El fichero `credentials.yml` define las credenciales referenciando variables:

```yaml
controller_credentials:
  - name: "Credencial Linux Produccion"
    organization: "Linux"
    credential_type: "Machine"
    inputs:
      username: "ansible-svc"
      password: "{{ vault_machine_password }}"    # ← referencia, no el valor
```

El fichero `secrets.yml` contiene los valores reales, cifrados:

```
$ANSIBLE_VAULT;1.1;AES256
63643434646332303036643830313162646166376162303031333963363366363433
3235373136346261333361636333643033326165656239346666356636660a6632...
```

**Flujo día a día:**

| Paso | Quién | Qué hace |
|---|---|---|
| 1 | Dev | `ansible-vault edit secrets.yml` → añade/modifica un valor cifrado |
| 2 | Dev | Edita `credentials.yml` → añade la definición con `{{ vault_* }}` |
| 3 | Dev | `git commit` + `git push` |
| 4 | Pipeline | Ejecuta el playbook con `--vault-password-file` (password desde GitHub Secret) |
| 5 | AAP | Recibe las credenciales con valores reales, los almacena como `$encrypted$` |

**Ejecutarlo desde AAP directamente:**

Si en vez de GitHub Actions se quiere ejecutar desde AAP:

1. Crear un proyecto apuntando al repo Git (con `scm_update_on_launch: true`)
2. Crear una credencial de tipo **Vault** con la vault password
3. Crear un JT con el playbook y asociarle la credencial Vault
4. Lanzar el JT — AAP descifra `secrets.yml` automáticamente

**Qué se ve en cada capa:**

| Dónde | Qué se ve |
|---|---|
| Git | `secrets.yml` cifrado, `credentials.yml` con `{{ vault_* }}` |
| Logs del pipeline / AAP | `no_log: true` — valores ocultos |
| AAP (credenciales almacenadas) | `$encrypted$` |

**Ventajas:**
- No requiere infraestructura adicional
- Todo vive en el repo (cifrado)
- Se integra directamente con AAP (credencial tipo Vault)
- Ansible Vault es una herramienta que ya conocéis

**Limitaciones:**
- La vault password es un secreto compartido — hay que gestionarla
- Rotar la vault password requiere re-cifrar `secrets.yml`
- No hay control de acceso granular — quien tenga la vault password descifra todo

---

### Opción 2 — Secretos del CI/CD (GitHub Secrets / GitLab Variables)

Los valores sensibles no están en el repo en absoluto. Viven como secretos del sistema de CI/CD y se inyectan como variables al ejecutar el playbook.

**Cómo funciona:**

```
Repositorio Git
├── credentials.yml          ← definiciones con {{ variables }} (legible, sin secretos)
├── aplicar_credenciales.yml ← playbook
└── .gitignore

GitHub Secrets (fuera del repo)
├── MACHINE_PASSWORD = "SuperSecretPassword123!"
├── VCENTER_PASSWORD = "vC3nt3r!Admin"
└── CONTROLLER_PASSWORD = "password"
```

Pipeline en GitHub Actions:

```yaml
- name: Aplicar credenciales
  run: |
    ansible-playbook aplicar_credenciales.yml \
      -e "vault_machine_password=${{ secrets.MACHINE_PASSWORD }}" \
      -e "vault_vcenter_password=${{ secrets.VCENTER_PASSWORD }}" \
      -e "controller_password=${{ secrets.CONTROLLER_PASSWORD }}"
```

**Ventajas:**
- Ningún secreto en el repo (ni siquiera cifrado)
- Los secretos se gestionan desde la UI de GitHub/GitLab
- Rotación de secretos sin tocar el repo

**Limitaciones:**
- Los secretos están acoplados al sistema de CI/CD — si se cambia de GitHub a GitLab hay que migrarlos
- No se puede ejecutar el playbook localmente sin configurar las variables a mano
- No se puede ejecutar directamente desde AAP (los secretos están en GitHub, no en el repo)

---

### Opción 3 — Gestor de secretos externo (HashiCorp Vault, CyberArk, Azure Key Vault)

Los valores sensibles viven en un gestor de secretos dedicado. El playbook de CaC los consulta en runtime con un lookup plugin y los pasa a AAP.

**Cómo funciona:**

El playbook, al ejecutarse, se autentica contra el vault externo (con un token, AppRole, etc.), extrae el valor del secreto y se lo pasa a AAP como haría con cualquier otra variable. AAP recibe el valor y lo almacena como `$encrypted$`.

```yaml
controller_credentials:
  - name: "Credencial Linux Produccion"
    credential_type: "Machine"
    inputs:
      username: "ansible-svc"
      password: "{{ lookup('hashi_vault', 'secret=aap/data/credentials/linux:password',
                   url='https://vault.mapfre.local',
                   token=hashi_token) }}"
```

La variable `hashi_token` (el token de autenticación contra el vault) es el **único secreto que hay que gestionar** — puede venir de Ansible Vault (`secrets.yml`) o de un GitHub Secret. Con él, el playbook se autentica contra HashiCorp Vault y extrae todos los demás valores.

```
┌──────────┐   1. se autentica   ┌──────────────┐
│ Playbook │   con hashi_token   │ HashiCorp    │
│ CaC      │ ──────────────────→ │ Vault        │
│          │←──────────────────│              │
│          │   2. recibe valor   └──────────────┘
│          │
│          │   3. crea credencial con el valor
│          │──────────────→ AAP ($encrypted$)
└──────────┘
```

El secreto **no está en git** — el playbook lo busca en vivo cada vez que se ejecuta. AAP lo almacena cifrado. Si alguien rota el secreto en el vault, hay que **re-ejecutar el playbook de CaC** para que AAP reciba el nuevo valor.

**Ventajas:**
- Los secretos no están ni en el repo ni en el CI/CD
- Gestión centralizada con auditoría y políticas de acceso
- Control de acceso granular (quién puede leer qué secreto)

**Limitaciones:**
- Requiere infraestructura de vault desplegada y mantenida
- El playbook necesita autenticarse contra el vault (token, AppRole, etc.)
- Si rotan el secreto en el vault, hay que re-ejecutar el CaC para actualizar AAP

---

### Opción 4 — AAP External Credential Lookup

Esta opción es fundamentalmente diferente a las tres anteriores. Aquí **AAP no almacena el valor del secreto**. En su lugar, AAP sabe dónde buscarlo y lo consulta en vivo cada vez que un Job Template lo necesita.

AAP soporta nativamente estos gestores:

- HashiCorp Vault Secret Lookup
- CyberArk Central Credential Provider
- CyberArk Conjur Secrets Manager
- Microsoft Azure Key Vault
- AWS Secrets Manager
- Thycotic DevOps Secrets Vault

**Cómo funciona:**

Se configuran dos credenciales en AAP:

1. Una credencial de tipo **"HashiCorp Vault Secret Lookup"** (o el gestor que uséis) con la conexión al vault.
2. La credencial real (ej: Machine) que en vez de tener el valor almacenado, tiene una referencia a la credencial de lookup y la ruta del secreto en el vault.

```
Cada vez que un JT se lanza:

┌──────────┐   1. JT necesita    ┌─────────┐   2. AAP consulta   ┌──────────────┐
│ Usuario  │──────────────────→ │  AAP    │──────────────────→ │ HashiCorp    │
│ lanza JT │                    │         │←──────────────────│ Vault        │
└──────────┘                    │         │   3. valor         └──────────────┘
                                │         │
                                │         │   4. inyecta al job
                                │         │──→ playbook ejecuta con el valor
                                └─────────┘

AAP NO almacena el valor. Lo busca, lo usa y lo descarta.
```

**¿Y cómo se configura esto desde CaC?**

Hay un problema de "huevo y gallina": para que AAP consulte el vault, primero necesitas crear la credencial de lookup con la conexión al vault. Esa credencial de lookup **sí tiene un valor sensible** (el token de HashiCorp). Así que el flujo en CaC es:

1. La **credencial de lookup** (conexión al vault) se crea con el token cifrado en Ansible Vault o GitHub Secret — es el **único secreto que gestionas tú**
2. Las **credenciales reales** (Machine, vCenter, etc.) se vinculan a la credencial de lookup — sin valores sensibles, solo la referencia al path en el vault

```yaml
controller_credentials:
  # Paso 1: credencial de conexión al vault (el único secreto que gestionas)
  - name: "HashiCorp Vault"
    credential_type: "HashiCorp Vault Secret Lookup"
    inputs:
      url: "https://vault.mapfre.local"
      token: "{{ vault_hashi_token }}"     # ← viene de Ansible Vault o GitHub Secret
      api_version: "v2"

  # Paso 2: credencial real — sin password, vinculada al lookup
  - name: "Credencial Linux Produccion"
    credential_type: "Machine"
    inputs:
      username: "ansible-svc"
    # El campo password NO se define aquí
```

Después de crear ambas credenciales, se vinculan desde la UI de AAP (o vía API):

**Credencial Linux Produccion → campos input source → password → seleccionar "HashiCorp Vault" → path: `aap/data/credentials/linux` → key: `password`**

A partir de ese momento, cada vez que un JT use "Credencial Linux Produccion", AAP va al vault, saca el valor y lo inyecta al job.

**Diferencia clave respecto a la opción 3:**

| | Opción 3 (lookup en playbook) | Opción 4 (AAP External Lookup) |
|---|---|---|
| **Quién busca el secreto** | El playbook de CaC | AAP en cada ejecución del JT |
| **Cuándo** | Al ejecutar el CaC | Cada vez que se lanza un job |
| **AAP almacena el valor** | Sí (`$encrypted$`) | **No** — lo busca en vivo |
| **Rotación de secretos** | Hay que re-ejecutar el CaC | **Transparente** — AAP lee siempre el último valor |
| **Si el vault cae** | No afecta (AAP ya tiene el valor) | **Los jobs fallan** (no puede resolver la credencial) |

**Ventajas:**
- Los valores nunca pasan por CaC ni por Git ni se almacenan en AAP
- Rotación de secretos completamente transparente — se rota en el vault, AAP usa el nuevo valor sin tocar nada
- Auditoría completa en el gestor de secretos

**Limitaciones:**
- Requiere un gestor de secretos desplegado
- La configuración del external credential se hace en AAP, no en CaC
- Dependencia en runtime del vault externo (si cae, los jobs fallan)

---

## Comparativa

| | Ansible Vault | CI/CD Secrets | Vault externo | AAP External Lookup |
|---|---|---|---|---|
| **Secretos en Git** | Sí (cifrados) | No | No | No |
| **Infraestructura adicional** | Ninguna | Ninguna | Gestor de secretos | Gestor de secretos |
| **Ejecutable desde AAP** | Sí (credencial tipo Vault) | No directamente | Sí (lookup plugin) | Sí (nativo) |
| **Ejecutable desde CLI local** | Sí | Requiere configurar vars | Sí (con acceso al vault) | N/A |
| **Rotación de secretos** | Re-cifrar + commit | Cambiar en UI de CI/CD | Automática en el vault | Automática en el vault |
| **Complejidad** | Baja | Baja | Media-alta | Media |
| **Control de acceso** | Todo o nada (vault password) | Por secreto en GitHub | Granular (políticas) | Granular (políticas) |
| **Validado en lab** | Sí | Sí | No | No |

---

## Recomendación

| Situación | Opción recomendada |
|---|---|
| Ya usáis CaC + GitHub Actions y queréis algo rápido | **Ansible Vault** |
| Queréis que ningún secreto esté en el repo, ni cifrado | **CI/CD Secrets** (GitHub Secrets) |
| Tenéis o vais a desplegar un gestor de secretos (HashiCorp, CyberArk) | **Vault externo** o **AAP External Lookup** |
| Necesitáis rotación automática de secretos sin tocar CaC | **AAP External Lookup** |

Las opciones no son excluyentes. Se puede empezar con Ansible Vault hoy y migrar a un vault externo cuando la infraestructura esté lista, sin cambiar la estructura del repo — solo se cambian las referencias `{{ vault_* }}` por lookups.

---

## Próximos pasos

1. **¿Tenéis o tenéis previsto un gestor de secretos** (HashiCorp Vault, CyberArk, Azure Key Vault)? Si sí, las opciones 3 y 4 son las más robustas a largo plazo.

2. **¿Preferís que los secretos estén en el repo (cifrados) o fuera del repo?** Ansible Vault los pone cifrados en el repo. GitHub Secrets los saca del repo completamente.

3. Si queréis, podemos preparar una **demo** con vuestra estructura de CaC concreta.

---

Mario Castro Ginés  
Technical Account Manager — Red Hat
