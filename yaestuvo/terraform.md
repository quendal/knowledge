Tienes razón, disculpa la confusión. No necesito modificar archivos ni delegar nada para esto. 

La configuración de Terraform ya está lista para trabajar con tu perfil SSO. Simplemente usa la variable de entorno `AWS_PROFILE=yaestuvo` cuando ejecutes los comandos de Terraform.

Para desplegar el ambiente dev, estos son los pasos:

1. **Verifica tu sesión SSO:**
```bash
aws sts get-caller-identity --profile yaestuvo
```

2. **Crea los recursos de backend (S3 y DynamoDB para el state):**
```bash
AWS_PROFILE=yaestuvo aws s3 mb s3://yaestuvo-terraform-state-dev --region us-east-1
AWS_PROFILE=yaestuvo aws s3api put-bucket-versioning --bucket yaestuvo-terraform-state-dev --versioning-configuration Status=Enabled

AWS_PROFILE=yaestuvo aws dynamodb create-table \
  --table-name yaestuvo-terraform-locks-dev \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

3. **Crea la KMS key y actualiza terraform.tfvars con el ARN**

4. **Despliega las tablas:**
```bash
cd infrastructure/envs/dev
AWS_PROFILE=yaestuvo terraform init
AWS_PROFILE=yaestuvo terraform plan
AWS_PROFILE=yaestuvo terraform apply
```

Si la sesión SSO vence en cualquier momento, ejecutas `aws sso login --profile yaestuvo` y continuamos.

¿Quieres que verifique tu conexión ahora o prefieres seguir con otras tareas de desarrollo?
