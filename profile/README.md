## Despliegue y Operación de la Aplicación de Microservicios

Esta guía detalla los pasos necesarios para desplegar y operar la aplicación de microservicios en un entorno AWS, utilizando GitHub Actions para la automatización CI/CD y ArgoCD para el despliegue continuo.

### Pre-requisitos

1.  **Cuenta de AWS o Laboratorio AWS:** Necesitarás acceso a un entorno AWS. Este proyecto fue desarrollado y probado utilizando un laboratorio de AWS (como AWS Learner Lab).
2.  **Credenciales de AWS CLI:** Al iniciar tu sesión en la cuenta o laboratorio de AWS, obtén las credenciales temporales de la CLI:
    *   `AWS_ACCESS_KEY_ID`
    *   `AWS_SECRET_ACCESS_KEY`
    *   `AWS_SESSION_TOKEN`
3.  **Repositorios del Proyecto:** Asegúrate de tener acceso a los siguientes repositorios:
    *   `ecommerce-microservice-backend-app` (Backend)
    *   `ecommerce-frontend-web-app` (Frontend)
    *   `ecommerce-chart` (Charts de Helm para ArgoCD)
    *   `infrastructure` (Código de Terraform para la infraestructura AWS)

### Configuración Inicial de Secretos de GitHub

Las credenciales de AWS obtenidas deben configurarse como "Secrets" en cada uno de los repositorios de GitHub mencionados anteriormente.

1.  Ve a cada repositorio en GitHub.
2.  Navega a `Settings` > `Secrets and variables` > `Actions`.
3.  Crea los siguientes secrets con los valores correspondientes de tus credenciales de AWS:
    *   `AWS_ACCESS_KEY_ID`
    *   `AWS_SECRET_ACCESS_KEY`
    *   `AWS_SESSION_TOKEN`
    *   Adicionalmente, otros secrets como `TF_REGION`, `TF_ECR_NAME`, y `CHARTS_REPO_TOKEN` (un Personal Access Token de GitHub para la comunicación entre pipelines) ya vienen por defecto pero segun se requiera serán necesarios y deben configurarse según las necesidades de las pipelines. Consulta la configuración de cada pipeline para la lista completa de secrets requeridos.

### Paso 1: Despliegue de la Infraestructura Base en AWS

La infraestructura (VPC, EKS, ECR, etc.) se gestiona mediante Terraform.

1.  **Repositorio:** `infrastructure`
2.  **Pipeline:** Ejecuta la pipeline `deploy-infra.yaml`.
    *   Esta pipeline aplicará la configuración de Terraform para crear o actualizar la infraestructura necesaria en AWS.
    *   Espera a que la pipeline finalice exitosamente. Esto asegura que tu clúster de Kubernetes (EKS), el registro de contenedores (ECR) y otros recursos estén listos.

### Paso 2: Despliegue Inicial de las Imágenes de la Aplicación

Una vez la infraestructura esté lista, debes construir y subir las imágenes Docker de tus aplicaciones al ECR.

1.  **Despliegue de Microservicios Backend:**
    *   **Repositorio:** `ecommerce-microservice-backend-app`
    *   **Pipeline:** Ejecuta la pipeline de despliegue principal (e.g., `deploy-and-trigger.yml`).
    *   **Ramas y Namespaces:**
        *   **Para un despliegue completo inicial en todos los entornos/namespaces (dev, staging, master/prod):** Ejecuta la pipeline manualmente para cada rama de entorno que tengas configurada (ej. `dev`, `staging`, `master`). Cada ejecución construirá las imágenes y las etiquetará apropiadamente, y luego activará la actualización del chart de Helm para el entorno correspondiente.
        *   **Para un despliegue solo en `master` (producción):** Ejecuta la pipeline seleccionando la rama `master`.

2.  **Despliegue de la Aplicación Frontend:**
    *   **Repositorio:** `ecommerce-frontend-web-app`
    *   **Pipeline:** Ejecuta la pipeline de despliegue principal de este repositorio.
    *   **Ramas y Namespaces:** Sigue la misma lógica que para el backend. Ejecuta la pipeline para cada rama de entorno deseada.

**Nota sobre el Despliegue Inicial:** El proceso de "ejecutar 3 veces" es una estrategia para poblar inicialmente diferentes namespaces o entornos si tu flujo de trabajo está configurado para manejar ramas (`dev`, `staging`, `master`) que mapean a diferentes configuraciones o namespaces en Kubernetes. Si solo usas un entorno principal (ej. `master`), una sola ejecución en esa rama es suficiente.

Una vez que las imágenes estén en ECR y los charts de Helm se hayan actualizado a través de sus pipelines, ArgoCD (que debe estar configurado en tu clúster EKS y apuntando al repositorio `ecommerce-chart`) detectará los cambios en el repositorio de charts y desplegará/actualizará automáticamente las aplicaciones en Kubernetes.

### Paso 3: Flujo de Desarrollo y Despliegue Continuo (Nuevas Features y Cambios)

Una vez que la aplicación está en funcionamiento, el siguiente flujo se aplica para introducir cambios o nuevas funcionalidades:

1.  **Desarrollo de Features:**
    *   Crea una nueva rama (feature branch) a partir de la rama de desarrollo principal (e.g., `dev`).
    *   Realiza los cambios de código necesarios en el backend (`ecommerce-microservice-backend-app`) o frontend (`ecommerce-frontend-web-app`).

2.  **Pull Request (PR) y Chequeo de Código:**
    *   Al finalizar el desarrollo de la feature, crea un Pull Request (PR) desde tu feature branch hacia la rama de integración (e.g., `dev`).
    *   Al crear el PR, una pipeline de "chequeo" o "validación" (configurada para ejecutarse en `pull_request`) se ejecutará automáticamente. Esta pipeline puede incluir pasos como compilación, pruebas unitarias, análisis de código estático, etc.
    *   Si la pipeline de chequeo es exitosa, indica que el código es seguro para ser integrado.

3.  **Merge del PR y Despliegue Automático:**
    *   Una vez que el PR es revisado y aprobado, haz "merge" del PR en la rama de integración (e.g., `dev`).
    *   Al hacer merge, la pipeline de despliegue principal (la misma que se usó en el Paso 2, pero esta vez activada por un `push` a la rama `dev`) se ejecutará automáticamente.
        *   Construirá las nuevas imágenes Docker con los cambios.
        *   Subirá las imágenes al ECR con nuevas etiquetas.
        *   **Activará la pipeline en el repositorio `ecommerce-chart`**.

4.  **Actualización del Chart y Sincronización de ArgoCD:**
    *   La pipeline en el repositorio `ecommerce-chart` (activada por el paso anterior) modificará el archivo `values-master.yaml` (o el archivo de values correspondiente a la rama/entorno) para apuntar a las nuevas etiquetas de imagen en ECR.
    *   Commit y push de estos cambios se realizarán automáticamente en el repositorio `ecommerce-chart`.
    *   **ArgoCD**, que está monitoreando el repositorio `ecommerce-chart`, detectará estos cambios.
    *   ArgoCD aplicará automáticamente los cambios en el clúster de Kubernetes, actualizando los deployments para usar las nuevas imágenes.

5.  **Promoción a Otros Entornos (Staging, Producción):**
    *   El proceso para promover cambios de `dev` a `staging`, y luego de `staging` a `master` (producción), típicamente involucra la creación de Pull Requests entre estas ramas.
    *   Cada merge a una rama de entorno superior (`staging`, `master`) activará su respectiva pipeline de despliegue, que a su vez actualizará el chart y será desplegado por ArgoCD en el entorno correspondiente.

Este flujo automatizado asegura que los cambios se integren, prueben y desplieguen de manera consistente y eficiente en la nube.
