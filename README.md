# Learn Terraform State Management

This is a companion repository to the Terraform State Management tutorial. Follow along on [Learn](https://learn.hashicorp.com/tutorials/terraform/state-cli?in=terraform/state)


``` shell
# 初期化
terraform init

terraform apply

# State内のリソース表示
terraform show
# リソースの一覧
terraform state list
```

## -replaceで再作成
``` shell
# -replaceを付与すると再作成を強制
terraform plan -replace="aws_instance.example"
terraform apply -replace="aws_instance.example"

# リソースを別なStateファイルに移動
cd new_state
cp ../terraform.tfvars .
terraform init
terraform apply
# Stateの移動
terraform state mv -state-out=../terraform.tfstate aws_instance.example_new aws_instance.example_new

cd ../
# example_newが存在（new_stateから移動されていることを確認）
terraform state list
# example_newが破棄される（main.tfにexample_newの定義がないため、example_newリソースを破棄する計画になっている。）
terraform plan
```

### main.tfに定義コピー
``` terraform
resource "aws_instance" "example_new" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.sg_8080.id]
  user_data              = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y apache2
              sed -i -e 's/80/8080/' /etc/apache2/ports.conf
              echo "Hello World" > /var/www/html/index.html
              systemctl restart apache2
              EOF
  tags = {
    Name = "terraform-learn-state-ec2"
  }
}
```

``` shell
# デプロイ
terraform apply

cd new_state
# リソースを移動したため削除対象がない
terrafrom destroy
```


``` shell
cd ../
# Stateからリソースを削除
terraform state rm aws_security_group.sg_8080
terraform state list
# セキュリティグループをStateに戻す
terraform import aws_security_group.sg_8080 $(terraform output -raw security_group)

```


``` shell
# インスタンスの削除
aws ec2 terminate-instances --instance-ids $(terraform output -raw instance_id)


# エラー発生
An error occurred (UnauthorizedOperation) when calling the TerminateInstances operation: You are not authorized to perform this operation. Encoded authorization failure message: eKXmx2rKa2iHItwbkaMOeiL-aIRfJV9C0nspAL3frgMFlN0CTet1vcrNXws09lxVrrFw56B4EnK9P-WQK1WVk_yyQGxHhW6bAU84m1im8Kjexd5CS56RVgdrN8MfC5OPjKJ6IUBLUYBu50fufH--1RD0-Nq7xeBztrcrAmsViNPRDpApj9fjQm2s-xZvc0SKBACwjH_IXgz8n_Q1fkMC2DyCo8E-HrZlCp2Jqy4kGlTNQHPK-tOD2zjvODtuXVJcU9BEnyqAHdT3VqK0d94j5Cg-w1JRCYuQiy0CuvqmC3xuDxrpOU40Tf7M8LoJiWfFuOpYkWVE4iBkj2y7CzXyeOr4puKwzG2SzmFf8_vg9fvGdugKlCPIt0_OGdFxDhd0dI62GC42DWT9A3C7qFoAPBjNcNcFuQ_8fZQbc2sXxKemxNhFu57LVGtDnyZ4LO1pX6q0MA-eJBtubxMN8WB-j4FL9h0efuOZhzPnJoeuCy1acwIVa8xGPwdA9ybpYdWz1p3XQ0WZxuMu7SP_SvSCdyI0XB5TauKUfuK_BUs_MSqvSvv913O_WGG0L29CivRgytB3x4gZe2iPL3TuRmloXYN3zhjQWkWfliqK0-lr3FA_JeV59-ASiPml54PQc-ucoaJlDZCkTeYr2QD-vgJuQBM2wjxLr6FpWaf-kiLcI3HifS1MlFlMPQOzQhYbyuakQPt_KsNTkwTsRdWVS1_dacA2qe93rUYcF8s4fQDAvuoBtyYHALmVnLwvDmDaelGDJtbWyz51i8NvHnZ8e3PGbgtt3Rp0iBF8cT5XI0cWWNTi1fs2y3QmdlK52RyhZgcaVQ

# エラー内容確認権限がないっぽい
aws sts decode-authorization-message --encoded-message 

# awsコマンドの実行権限がなかったようなので、defaultを指定
aws ec2 terminate-instances --instance-ids $(terraform output -raw instance_id) --profile default


terraform refresh
aws_security_group.sg_8080: Refreshing state... [id=sg-03537c4ad0c3947c3]
aws_instance.example_new: Refreshing state... [id=i-0a6ca7c1ec7659670]

terraform state list
data.aws_ami.ubuntu
aws_instance.example_new
aws_security_group.sg_8080

```
