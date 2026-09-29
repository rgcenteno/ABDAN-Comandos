# Guía rápida de comandos

## Generales

### Ver credenciales de quién hace la petición

```sh
aws sts get-caller-identity
```

## EC2

### Obtener metadatos de la instancia EC2 a la que estamos conectados

```sh
curl http://169.254.169.254/latest/meta-data
```

Obtener el instance id document. Devuelve datos como id de instancia, AZ o la región. Se adjunta la versión v2 que es compatible en máquinas actuales.

```sh
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` \
    && curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/dynamic/instance-identity/document
```

### Obtener datos de la instancia

En este caso obtendremos los módulos EBS

```sh
aws ec2 describe-instances \
--filter 'Name=tag:Name,Values=Processor' \
--query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.{VolumeId:VolumeId}'
```

### Cambiar tipo de instancia EC2 (necesario detener primero)

```sh
aws ec2 modify-instance-attribute --instance-type t3.large
```

### Políticas adjuntas a un usuario

```sh
aws iam list-attached-user-policies --user-name $user
```

### Políticas inline

```sh
aws iam list-user-policies --user-name $user
```

### Crear un grupo de seguridad

```sh
aws ec2 create-security-group \
--group-name MySG \
--description "Security group para base de datos" 
\--vpc-id $VPC_ID
```

### Establecer una regla de entrada a un Security Group

```sh
aws ec2 authorize-security-group-ingress \
--group-id $SECURITY_GROUP_ID \
--protocol tcp --port 3306 \
--source-group sg-0465e354b41c6e55e
```

### Comprobar regla de seguridad

```sh
aws ec2 describe-security-groups 
--query "SecurityGroups[*].[GroupName,GroupId,IpPermissions]" 
--filters "Name=group-name,Values='MySG'"
```

## S3

### Crear un bucket en una región específica

Versión s3api

```sh
aws s3api create-bucket \
 --bucket mi_bucket \
 --region eu-west-2 \
 --create-bucket-configuration LocationConstraint=eu-west-2
```

Versión s3

```sh
aws s3 mb s3://mi_bucket --region eu-west-2
```

### Activar control de versiones

```sh
aws s3api put-bucket-versioning \
 --bucket rgcenteno-s3api-0809 \
 --region eu-west-2 \
 --versioning-configuration Status=Enabled
```

### Borrar un bucket

Versión s3api. Sólo funciona con buckets vacíos.

```sh
aws s3api delete-bucket \
 --bucket rgcenteno-s3api-0809
```

Versión s3. Necesario `--force` si no está vacío el bucket.

```sh
aws s3 rb s3://rgcenteno-s3api-0809 --force
```

### Mostrar buckets en cuenta

```sh
aws s3 ls
```

### Mostrar buckets en una región

```sh
aws s3 ls --region eu-west-2
```

### Listar contenido de bucket

```sh
aws s3 ls s3://amzn-s3-demo-bucket
```

### Copiar un fichero local en S3

```sh
aws s3 cp file.txt s3://my-bucket/folder/file.txt
```

Si queremos añadir la clase de almacenamiento.

```sh
aws s3 cp file.txt s3://my-bucket/folder/file.txt \
    --storage-class INTELLIGENT_TIERING 
```

Clases soportadas:

- `STANDARD`

- `STANDARD_IA` (estándar poco frecuente)

- `INTELLIGENT_TIERING`

- `ONZONE_IA` estándar poco frecuente zona única

- `GLACIER`

### Sincronizar objetos en un bucket determinado

Sincroniza en el directorio actual el contenido de la carpeta myprefix del bucket my-bucket. **Primer parámetro origen, segundo: destino***.

```sh
aws s3 sync s3://my-bucket/myprefix .
```

### Borrar elemento de un bucket

```sh
aws s3 rm s3://my-bucket/folder/file.txt
```

### Hacer un listado de  todas las versiones de un objeto

```sh
aws s3api list-object-versions \
    --bucket mi-nombre-de-bucket \
    --prefix mi-fichero-borrado.txt
```

### Recuperar en S3 con control de versiones un fichero eliminado

Primero obtenemos el VersionId del fichero

```sh
aws s3api list-object-versions \
--bucket mi-nombre-de-bucket \
--prefix mi-fichero-borrado.txt \
--query "DeleteMarkers[?IsLatest==\`true\`].VersionId" \
--output text
```

Luego restauramos el fichero en el s3

```sh
aws s3api delete-object \
--bucket mi-nombre-de-bucket \
--key mi-fichero-borrado.txt \
--version-id ID_DEL_MARCADOR_DE_BORRADO
```

### Descargar una versión específica de un fichero en un S3

```sh
aws s3api get-object \
--bucket rgcenteno-1009 \
--key files/file1.txt \
--version-id NPhUZHh1IzxN_QIKiX_wmG2zReZ6yLPE \
carpeta_local/nombre_fichero.txt
```

## EBS

### Crear un volumen

```sh
aws ec2 create-volume --size 80 \
 --avaliability-zone eu-west-2 \
 --volume-type gp3
```

### Obtener datos sobre un volumen

```sh
aws ec2 describe-volumes --volume-ids <mi_id>
```

### Asociar a una instancia

[Documentación AWS](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html) 

```sh
aws ec2 attach-volume --volume-id <id_vol> \
 --instance-id id_ec2 --device /dev/sdf
```

### Crear una instantánea de EBS

Para volúmenes raíz es recomendable parar la instancia antes de hacer la instantánea

```sh
aws ec2 create-snapshot --volume-id id_vol \
    --descripcion "Nueva instantánea"
```

### Copiar una instantánea a otra región

Las instantáneas se limitan al nivel de la región y se pueden ver únicamente en la región en la que se crearon. Si desea usar una instantánea en una región diferente o hacer una copia de ella en la misma región, puede usar el comando `copy-snapshot` de la CLI de AWS para copiarla.

```sh
aws ec2 copy-snapshot --region us-east-1 \
    --source-region us-west-2 \
    --source-snapshot-id snap-id-123467
    --description "This is a copy"
```

### Comprobar datos de un snapshot

```sh
aws ec2 describe-snapshots \
    --snapshot-ids snap-id-abcde
    --region us-east-1 
```

### Restaurar un snapshot a un nuevo volumen

```sh
aws ec2 create-volume \
  --snapshot-id snap-xxxxxxxx \
  --availability-zone us-east-1 \
  --volume-type gp3
  --size 80
```

### Desasociar un volumen a una máquina

```sh
aws ec2 detach-volume --volume-id vol-antiguo
```

```sh
aws ec2 attach-volume \
  --volume-id vol-restaurado \
  --instance-id i-xxxxxxxx \
  --device /dev/xvda
```

### Restaurar un volumen conectado a una instancia EC2

Debemos restaurar la imagen a un nuevo volumen, luego tendríamos dos opciones:

* [Desasociar el volumen antiguo](#desasociar-un-volumen-a-una-mquina) y [asociar el volumen restaurado](#asociar-a-una-instancia)

* [Asociar el volumen restaurado](#asociar-a-una-instancia) a la máquina EC2 y copiar los ficheros que queremos recuperar

## VPC

### Crear una Subnet

```shell
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.200.2.0/23 --avaliability-zone us-east-1a
```

## RDS

### Crear un grupo de subred

Amazon RDS te obliga a seleccionar al menos dos subredes en dos Zonas de Disponibilidad (AZ) diferentes. Esto es obligatorio incluso aunque vayamos a desplegar un RDS Single-AZ para garantizar la **alta disponibilidad** y la **tolerancia a fallos** de tu base de dato.

```sh
aws rds create-db-subnet-group \
--db-subnet-group-name "MyDB Subnet Group" \
--db-subnet-group-description "DB subnet for My DB" \
--subnet-ids subnet-0ea4d8829de8865a3 subnet-066542fb7e0844a29 \
--tags "Key=Name,Value= MyDatabaseSubnetGroup"
```

### Crear una instancia RDS

```sh
aws rds create-db-instance \
--db-instance-identifier MyDBInstance \
--engine mariadb \
--db-instance-class db.t3.micro \
--allocated-storage 20 \
--availability-zone us-east-1a \
--db-subnet-group-name "MyDB Subnet Group" \
--vpc-security-group-ids sg-0693fe00de879ccea \
--no-publicly-accessible \
--master-username root --master-user-password 'password'
```

### Consultar estado de instancia RDS

```sh
aws rds describe-db-instances \
--db-instance-identifier MyDBInstance \
--query "DBInstances[*].[Endpoint.Address,AvailabilityZone,PreferredBackupWindow,BackupRetentionPeriod,DBInstanceStatus]"
```

## Amazon DLM (Data Lifecycle Manager)

Herramienta para automatizar la creación, retención y eliminado de instantáneas

### Creación de rol de IAM necesario para que DLM funcione

```sh
aws dlm create-default-role
```

### Crear una política de ciclo de vida

```sh
aws dlm --create-lifecycle-policy \
    --description "Política de ciclo de vida" \
    --state ENABLED \
    --execution-role-arn \
    arn:aws:iam:11111111:role/AWSDataLifeCycleManagerDefaultRole \
    --policy-details file://policyDetails.json
```

**Fichero policyDetails.json**

```json
{
"ResourceTypes": [
      "VOLUME"
   ],
   "TargetTags"    : [
      {
         "Key"        : "name",
         "Value"    : "production"
      }
   ],
   "Schedules"    :[
      {
         "Name"    : "DailySnapshots",
         "TagsToAdd": [
            {
               "Key"    : "type",
               "Value": "myDailySnapshot"
            }
         ],
         "CreateRule"    : {
            "Interval"    : 24,
            "IntervalUnit": "HOURS",
            "Times"    : [
               "03:00"
            ]
         },
         "RetainRule"    : {
            "Count":5
         },
         "CopyTags": false 
      }
   ]
}
```
