# aws-privatelink-private-instance-ssm
Plantillas de CloudFormation para desplegar y acceder de forma segura a instancias EC2 privadas mediante VPC Endpoints y Systems Manager

# Guía de Despliegue Completo

Para desplegar la infraestructura privada usando CloudFormation desde tu computadora, asegúrate de tener actualizado tu perfil de AWS en tu terminal (`aws login --profile remotepc`). Debes estar ubicado en el directorio raíz de este repositorio.


No olvides usar el flag `--profile` o como hayas definido el perfil pero en mi caso es `remotepc` en cada comando.

---

## 1. Despliegue de la VPC Privada

### Opción 1: Desplegar con los valores por defecto
```bash
aws cloudformation create-stack \
  --stack-name stack-private-vpc \
  --template-body file://vpc/template.yaml \
  --profile remotepc
```
---

## 2. Despliegue de VPC Endpoints (PrivateLink para Fleet Manager)

Una vez lista la VPC, despliega los Endpoints requeridos usando los IDs provistos. Si en tu caso la VPC usa otro prefijo CIDR diferente a `10.0.0.0/24`, ajústalo en el comando para que las reglas internas del Security Group te permitan el tráfico seguro y fluído.

```bash
aws cloudformation create-stack \
  --stack-name stack-private-vpce \
  --template-body file://vpce/template.yaml \
  --profile remotepc \
  --parameters \
      ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxx \
      ParameterKey=VpcCidrBlock,ParameterValue=x.x.x.x/xx \
      ParameterKey=SubnetIds,ParameterValue=subnet-xxxxxxxxxxxxx
```

---

## 3. Despliegue del VPC Endpoint para S3 (Gateway o Interface)

Puedes escoger conectarte a S3 implementando un Endpoint tipo Gateway o un Endpoint tipo Interface. Escoge solo una de las siguientes opciones según tu arquitectura:

### Opción A: Despliegue del VPC Endpoint tipo Gateway (Recomendado)
Este endpoint no requiere una subred particular ni Security Groups, simplemente necesita el ID de la **Route Table** (`RouteTableId`) de tu subred privada para configurarle la ruta directamente.

```bash
aws cloudformation create-stack --stack-name stack-private-vpce-s3-gw --template-body file://vpce-s3-gateway/template.yaml --profile remotepc --parameters ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxx ParameterKey=RouteTableId,ParameterValue=rtb-xxxxxxxxxxxxx
```

### Opción B: Despliegue del VPC Endpoint tipo Interface
A diferencia del Gateway, este endpoint utiliza _Network Interfaces (ENIs)_ situados en tus subredes. Internamente crea su propio **Security Group** para permitir acceso seguro HTTPS (443) hacia S3, por lo que adicionalmente a tu VPC y Subredes (`SubnetIds`), necesitas pasarle el valor de red del bloque CIDR de tu VPC (`VpcCidrBlock`).

```bash
aws cloudformation create-stack --stack-name stack-private-vpce-s3-if --template-body file://vpce-s3-interface/template.yaml --profile remotepc --parameters ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxx ParameterKey=VpcCidrBlock,ParameterValue=x.x.x.x/xx ParameterKey=SubnetIds,ParameterValue=subnet-xxxx1,subnet-xxxx2
```

---

## 4. Despliegue de la Instancia EC2 (Windows / Linux)

Despliega tu instancia. Notarás que la plantilla de forma predeterminada usará una AMI de **Linux**, pero si quieres usar **Windows**, ajusta el parámetro `AmiId` con el valor soportado. Al ejecutarse este comando, **es indispensable** mandarle el argumento `--capabilities CAPABILITY_IAM` ya que el template crea un nuevo "IAM Role" e "Instance Profile" en tu cuenta para que el SSM Agent tenga permisos.

También estamos incluyendo el parámetro `BucketName` para que la instancia tenga acceso al bucket que desplegarás.

```bash
aws cloudformation create-stack \
  --stack-name stack-private-instance \
  --template-body file://instance/template.yaml \
  --capabilities CAPABILITY_IAM \
  --profile remotepc \
  --parameters \
      ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxxx \
      ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxxxxxxx \
      ParameterKey=BucketName,ParameterValue=shared-data
```

---

## 5. Conectarse a la Instancia (y Contraseñas)

Dependiendo del sistema operativo que hayas desplegado en el paso 4, el método de acceso será muy distinto:

### ▶ Si usaste la AMI de Linux (Predeterminada)
No es necesario establecer ni usar ninguna contraseña. Puedes conectarte directamente de forma segura buscando tu instancia en la consola de AWS, haciendo clic en **Connect** y seleccionando la pestaña **Session Manager**. Esto te abrirá una terminal shell (`bash`) en tu navegador autenticada por tu propia sesión de IAM.

### ▶ Si usaste la AMI de Windows (RDP / Fleet Manager)
Al no haber asignado un Key Pair en la plantilla EC2, no conocemos la clave base generada con la AMI y el RDP sí nos la pedirá. Usa el siguiente comando apoyándote en SSM para forzar tu propia contraseña y así habilitar el acceso. 

Sustituye el valor `EL_ID_DE_TU_INSTANCIA` por el ID real y coloca una contraseña de tu gusto:

```bash
aws ssm send-command \
  --instance-ids "EL_ID_DE_TU_INSTANCIA" \
  --document-name "AWS-RunPowerShellScript" \
  --parameters '{"commands":["net user Administrator \"TuNuevaContrasenaSegura123!\""]}' \
  --profile remotepc
```

Una vez que este comando devuelva el objeto confirmando el envío exitoso, ve a **Fleet Manager** en tu consola de Systems Manager para abrir el RDP en tu navegador seleccionando **User credentials**, usando el usuario `Administrator` y la llave que acabas de forzar a nivel de comando.

---

## 6. Despliegue del Bucket S3

Despliega el bucket S3 privado mediante este comando de una sola línea. Puedes cambiar el valor de `shared-data` si deseas otro prefijo (asegúrate de que coincida con el provisto a la instancia EC2):

```bash
aws cloudformation create-stack --stack-name stack-private-bucket --template-body file://bucket/template.yaml --parameters ParameterKey=BucketName,ParameterValue=shared-data --profile remotepc
```
