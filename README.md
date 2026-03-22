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

## 3. Despliegue de la Instancia EC2 Windows

Despliega tu instancia. Al ejecutarse este comando, **es indispensable** mandarle el argumento `--capabilities CAPABILITY_IAM` ya que el template crea un nuevo "IAM Role" e "Instance Profile" en tu cuenta para que el SSM Agent tenga permisos.

```bash
aws cloudformation create-stack \
  --stack-name stack-private-instance \
  --template-body file://instance/template.yaml \
  --capabilities CAPABILITY_IAM \
  --profile remotepc \
  --parameters \
      ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxxx \
      ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxxxxxxx
```

---

## 4. Establecer la Contraseña de Administrador (RDP)

Al no haber asignado un Key Pair en la plantilla EC2, no conocemos la clave base generada para Windows. Usa el siguiente comando apoyándote en SSM para forzar tu propia contraseña y así habilitar el acceso por Fleet Manager. 

Sustituye el valor `EL_ID_DE_TU_INSTANCIA` por el ID real y coloca una contraseña de tu gusto:

```bash
aws ssm send-command \
  --instance-ids "EL_ID_DE_TU_INSTANCIA" \
  --document-name "AWS-RunPowerShellScript" \
  --parameters '{"commands":["net user Administrator \"TuNuevaContrasenaSegura123!\""]}' \
  --profile remotepc
```

Una vez que este comando devuelva el objeto confirmando el envío, asume que fue exitoso y ve a **Fleet Manager** en tu consola para conectarte usando **User credentials** con el usuario `Administrator` y la llave que impusiste aquí.
