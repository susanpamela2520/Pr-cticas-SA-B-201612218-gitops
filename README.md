# Repositorio GitOps — Plataforma de Tickets

**Practica 8 · Software Avanzado 2S 2026 · Carne 201612218 · Seccion B**

Este repositorio es la **unica fuente de verdad** del estado del sistema en
el cluster. Nada se aplica a mano: ArgoCD reconcilia continuamente el
cluster con lo que hay aqui.

Repositorio de codigo (servicios, Dockerfiles, pipeline):
https://github.com/susanpamela2520/Pr-cticas-SA-B-201612218

---

## Como llega un cambio aqui

El pipeline del repositorio de codigo **no tiene acceso al cluster**. Su
capacidad maxima es abrir un Pull Request que modifica una sola linea: la
etiqueta de la imagen en `rollouts/api-gateway-rollout.yaml`.

```
tag v1.0.1 en el repo de codigo
   -> pipeline: lint, Trivy, build, SBOM, firma, verificacion
   -> Pull Request automatico en ESTE repositorio
   -> aprobacion humana
   -> ArgoCD detecta el cambio en main
   -> Argo Rollouts: canary 20% -> 50% -> 100%
```

---

## Estructura

| Carpeta | Contenido | Application de ArgoCD |
|---|---|---|
| `apps/` | Definiciones de Application | se aplican a mano, una vez |
| `rollouts/` | Rollout canary, analisis y servicios | `sa-platform-gateway` |
| `policies/` | Politicas de admision de Kyverno | `sa-platform-politicas` |
| `secrets/` | Secretos cifrados con Sealed Secrets | `sa-platform-gateway` |

---

## Aplicaciones registradas

| Nombre | Namespace destino | Ruta | Sincronizacion |
|---|---|---|---|
| `sa-platform-gateway` | `sa-prod` | `rollouts/` | automatica, con prune y selfHeal |
| `sa-platform-politicas` | `kyverno` | `policies/0*.yaml` | automatica, con prune y selfHeal |

`selfHeal: true` significa que cualquier cambio manual hecho con kubectl se
revierte automaticamente. Es deliberado: una modificacion no trazable no
debe sobrevivir.

---

## Entrega progresiva

`rollouts/api-gateway-rollout.yaml` define un canary de tres pasos:

| Paso | Trafico al canary | Pausa |
|---|---|---|
| 1 | 20 % | 2 min |
| 2 | 50 % | 2 min |
| 3 | 100 % | 1 min |

En cada paso corre el `AnalysisTemplate` de `rollouts/analysis-template.yaml`,
con tres metricas:

| Metrica | Umbral | failureLimit |
|---|---|---|
| `disponibilidad` | HTTP 200 obligatorio | 0 |
| `tasa-de-exito` | >= 99 % de 100 peticiones | 2 |
| `latencia-p95` | <= 500 ms | 2 |

Los umbrales se derivan de la linea base medida con k6 en la Practica 6
(0,01 % de error, p95 de 222 ms). La justificacion completa esta en los
comentarios del propio archivo.

Si alguna metrica no alcanza su umbral, Argo Rollouts revierte al estable
sin intervencion humana. El trafico maximo afectado es el 20 %, que es el
peso del primer paso.

---

## Politicas de admision

| Politica | Modo | Que exige |
|---|---|---|
| `disallow-latest-tag` | Enforce | Etiqueta explicita distinta de `latest` |
| `require-resource-limits` | Enforce | `requests` y `limits` de CPU y memoria |
| `require-run-as-nonroot` | Enforce | `runAsNonRoot: true` y sin escalada de privilegios |

Se aplican solo en namespaces etiquetados
`sa-platform/aplicar-politicas: bloquear`. En staging auditan sin bloquear.

`policies/pod-de-prueba-rechazado.yaml` viola las tres a la vez y sirve
para evidenciar el rechazo:

```bash
kubectl apply -f policies/pod-de-prueba-rechazado.yaml
```

Esta excluido de la Application, asi que ArgoCD no intenta aplicarlo.

---

## Secretos

Ningun secreto en texto plano vive aqui. Ver `secrets/README.md` para el
procedimiento con Sealed Secrets: la llave privada nunca sale del cluster,
por lo que el archivo cifrado es seguro en un repositorio publico.

---

## Comandos utiles

```bash
# Estado de las aplicaciones
kubectl get applications -n argocd

# Seguir una promocion paso a paso
kubectl argo rollouts get rollout api-gateway -n sa-prod --watch

# Historial de versiones desplegadas
kubectl argo rollouts history api-gateway -n sa-prod

# Resultado de los analisis
kubectl get analysisrun -n sa-prod

# Politicas activas
kubectl get clusterpolicy
```
