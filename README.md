# Laboratorio Serverless — OpenFaaS + Python
https://xrugamas.github.io/laboratorio-serverless/
Laboratorio de **Introducción al Cómputo en la Nube** (Unidad 3: Computación Serverless y modelo FaaS).

Función Serverless en Python (`python3-http` de OpenFaaS) que:

- con **GET** presenta un formulario web (nombre, apellido y correo del aula virtual);
- con **POST** genera un archivo **JSON descargable** con los datos, un token UUID único, la fecha UTC,
  información del entorno de ejecución (contenedor Linux) y una firma **SHA-256** del contenido.

La evidencia de la ejecución es el archivo `registro_<token>.json` en la raíz del repositorio.

## Estructura

```
laboratorio-serverless/
├── stack.yaml                      # definición de la función para OpenFaaS
├── registro-serverless/
│   ├── handler.py                  # código de la función
│   └── requirements.txt
├── template/python3-http/          # plantilla oficial (faas-cli template store pull python3-http)
├── build/registro-serverless/      # contexto de construcción de la imagen Docker
└── registro_<token>.json           # evidencia generada por el servicio
```

## Entorno utilizado

| Herramienta | Uso |
|---|---|
| WSL 2 | Motor de Docker Desktop en Windows 11 |
| Docker Desktop | Contenedores |
| Kubernetes (Docker Desktop, Kubeadm) | Orquestación local |
| Helm | Instalación de OpenFaaS CE (chart `openfaas/openfaas`) |
| OpenFaaS CLI (`faas-cli`) | Plantilla, creación y administración de la función |

## Cómo ejecutarlo

Requisitos: Docker Desktop en ejecución.

```powershell
# 1. Preparar el contexto de construcción (plantilla + función)
New-Item -ItemType Directory -Force -Path "build\registro-serverless\function" | Out-Null
Copy-Item "template\python3-http\Dockerfile","template\python3-http\index.py","template\python3-http\requirements.txt" "build\registro-serverless\"
Copy-Item "registro-serverless\*" "build\registro-serverless\function\" -Recurse -Force

# 2. Construir la imagen
docker build -t registro-serverless:v1 ./build/registro-serverless

# 3. Ejecutar la función (8081 porque el 8080 lo usa el Gateway de OpenFaaS)
docker run --rm -p 8081:8080 registro-serverless:v1
```

Abrir <http://localhost:8081>, completar el formulario y presionar **Generar archivo JSON**.

### OpenFaaS en Kubernetes

```powershell
helm repo add openfaas https://openfaas.github.io/faas-netes/
kubectl create namespace openfaas
kubectl create namespace openfaas-fn
helm upgrade --install openfaas openfaas/openfaas `
  --namespace openfaas `
  --set functionNamespace=openfaas-fn `
  --set generateBasicAuth=true `
  --set serviceType=ClusterIP
kubectl port-forward -n openfaas svc/gateway 8080:8080
```

Gateway en <http://127.0.0.1:8080> (usuario `admin`; la contraseña está en el secreto `basic-auth`
del namespace `openfaas`).

> **Nota:** OpenFaaS Community Edition solo despliega imágenes de registros públicos
> (`faas-cli deploy` con una imagen local responde *"the Community Edition license agreement only
> allows public images"*). Por eso, como indica la guía, la función se ejecuta con `docker run`
> sobre la imagen construida localmente. Para desplegarla en el Gateway habría que publicar la imagen
> en un registro público (por ejemplo Docker Hub) y apuntar `image:` de `stack.yaml` a ella.

## Comprobar la firma del JSON

La firma es el SHA-256 del registro **sin** el campo `firma_sha256`, serializado con claves ordenadas:

```python
import json, hashlib
r = json.load(open("registro_<token>.json", encoding="utf-8"))
firma = r.pop("firma_sha256")
calc = hashlib.sha256(json.dumps(r, sort_keys=True, ensure_ascii=False).encode("utf-8")).hexdigest()
print("Firma válida" if calc == firma else "Firma NO coincide")
```
