# Gestion de secretos con Sealed Secrets

Ningun secreto en texto plano vive en este repositorio.

## Como funciona

Sealed Secrets usa criptografia asimetrica. El controlador genera un par de
llaves dentro del cluster; la publica se usa para cifrar y **la privada
nunca sale del cluster**.

Eso significa que el archivo `sealed-secrets.yaml` de esta carpeta es
seguro de versionar en un repositorio publico: solo el controlador que
tiene la llave privada puede descifrarlo, y esa llave no esta aqui.

```
values-secrets.yaml (local, nunca se sube)
        |
        | kubeseal --cert pub-cert.pem
        v
sealed-secrets.yaml (cifrado, se versiona)
        |
        | el controlador lo descifra dentro del cluster
        v
Secret de Kubernetes (existe solo en el cluster)
```

## Generar el archivo cifrado

1. Instalar el controlador en el cluster:

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system
```

2. Instalar la herramienta `kubeseal` en la maquina local (Windows):

```bash
winget install BitnamiLabs.SealedSecrets.Kubeseal
```

3. Obtener el certificado publico del controlador:

```bash
kubeseal --fetch-cert --controller-name sealed-secrets --controller-namespace kube-system > pub-cert.pem
```

4. Crear el Secret **sin aplicarlo** y cifrarlo:

```bash
kubectl create secret generic sa-platform-secretos \
  --namespace sa-prod \
  --from-env-file=secretos.env \
  --dry-run=client -o yaml \
  | kubeseal --cert pub-cert.pem --format yaml > sealed-secrets.yaml
```

El `--dry-run=client` es lo que evita que el Secret en claro llegue al
cluster: se construye localmente, se cifra, y solo la version cifrada se
versiona.

5. Borrar el archivo intermedio:

```bash
rm secretos.env
```

## Verificar que no hay nada en claro

```bash
grep -riE "password|secret:|jwt.*[A-Za-z0-9]{20}" . --include="*.yaml"
```

El unico resultado esperado es el bloque `encryptedData` de
`sealed-secrets.yaml`, que es precisamente el contenido cifrado.
