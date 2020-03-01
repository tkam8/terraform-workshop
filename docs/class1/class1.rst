初めてのTerraform
================================================

**Goal**: 
----------------
ここではOSS版のTerraformを利用してAWS上に一つインスタンスを作り、それぞれのコンポーネントや用語について説明をします。

**Prerequisites**: 
------------------
- インターネットに接続可能なPC
- 管理者権限（Terraformをインストールするため）

**Terraform Installation**: 
------------------

#. Terraformがインストールされていない場合は[こちら](https://www.terraform.io/downloads.html)よりダウンロードをしてください。

#. ダウンロードしたらunzipして実行権限を付与し、パスを通します。下記はmacOSの手順です。

    .. code-block:: bash

    $ unzip terraform*.zip
    $ chmod + x terraform
    $ mv terraform /usr/local/bin
    $ terraform -version
    Terraform v0.12.21


.. note:: Windowsの場合はこちらのリンクをご参照ください。 https://dev.classmethod.jp/tool/try-terraform-on-windows/

#. 次に任意の作業用ディレクトリを作ります。

    .. code-block:: bash

    $ mkdir -p tf-workspace/hello-tf
    $ cd  tf-workspace/hello-tf

**Terraform Configuration Files**: 
------------------

#. 早速このフォルダにTerraformのコンフィグファイルを作ってみます。コンフィグファイルは`HashiCorp Configuration Language`というフレームワークを使って記述していきます。

`main.tf`と`vaiables.tf`という二つのファイルを作ってみます。`main.tf`はその名の通りTerraformのメインのファイルで、このファイルに記述されている内容がTerraformで実行されます。`variables.tf`は変数を定義するファイルです。各変数にはデフォルト値や型などを指定できます。

    .. code-block:: bash

        $ cat <<EOF > main.tf
        terraform {
            required_version = "~> 0.12"
        }

        provider "aws" {
            access_key = var.access_key
            secret_key = var.secret_key
        token      = var.session_token
            region     = var.region
        }

        resource "aws_instance" "hello-tf-instance" {
        ami = var.ami
        count = var.hello_tf_instance_count
        instance_type = var.hello_tf_instance_type

        tags = {
            Name = var.instance_name
        }
        }

        EOF

#. 次に`variables.tf`ファイルを作ります。

    .. code-block:: bash

        $ cat << EOF > variables.tf
        variable "access_key" {}
        variable "secret_key" {}
        variable "session_token" {}
        variable "region" {}
        variable "ami" {}
        variable "instance_name" {}
        variable "hello_tf_instance_count" {
            default = 1
        }
        variable "hello_tf_instance_type" {
            default = "t2.micro"
        }
        EOF

**Terraform Commands**: 
------------------

#. 二つのファイルができたらそのディレクトリ上でTerraformの初期化処理を行います。`init`処理ではステートファイルの保存先などのバックエンドの設定や必要ばプラグインのインストールを実施します。

    .. code-block:: bash

        $ terraform init

#. ここではAWSのプラグインがインストールされるはずです。

    .. code-block:: bash

        $  ls -R .terraform/plugins
        .terraform/plugins:
        linux_amd64

        .terraform/plugins/linux_amd64:
        lock.json                          terraform-provider-aws_v2.51.0_x4

#. 次に`plan`と`apply`を実施してインスタンスを作ってみましょう。AWSのコンソールまたはaws cliでインスタンスの状況確認しておいてください。

    .. code-block:: bash

    $ aws ec2 describe-instances --query "Reservations[*].Instances[*].{InstanceId:InstanceId,State:State,Name:Tags[?
    Key=='Name']|[0].Value}"


    .. note::　aws cliにログイン出来ていない場合、以下のコマンドでログインしてください。
    　　.. code-block:: bash

        　　 $ aws configure
            AWS Access Key ID [****************]: ****************
            AWS Secret Access Key [****************]: ****************
            Default region name [ap-southeast-1]:
            Default output format [json]:

#. `plan`はTerraformによるプロビジョニングの実行プランを計画します。実際の環境やステートファイルとの差分を検出し、どのリソースにどのような変更を行うかを確認することができます。`apply`はプランに基づいたプロビジョニングの実施をするためのコマンドです。

また、実行前に変数に値をセットする必要があります。方法としては

- `tfvars`というファイルの中で定義する
- `terraform apply -vars=***`という形でCLIの引数で定義する
- `TF_VAR_***`という環境変数で定義する
- Plan中に対話式で入力して定義する

がありますが、今回は環境変数でセットします。

    .. code-block:: bash

        $ export TF_VAR_access_key=************
        $ export TF_VAR_secret_key=************
        $ export TF_VAR_session_token=*********
        $ export TF_VAR_instance_name=<enteryourname>
        $ export TF_VAR_region=ap-southeast-1
        $ export TF_VAR_ami=ami-07ce5f60a39f1790e
        $ terraform plan
        $ terraform apply

#. Applyが終了するとAWSのインスタンスが一つ作られていることがわかるでしょう。AWSのコンソールまたはaws cliでインスタンスの状況確認してください。

    .. code-block:: bash
        $ aws ec2 describe-instances --query "Reservations[*].Instances[*].{InstanceId:InstanceId,State:State,Name:Tags[?
        Key=='Name']|[0].Value}"
        [
            {
                "InstanceId": "i-00918d5c9466da418",
                "State": {
                    "Code": 48,
                    "Name": "running"
                },
                "Name": "xxx"
            }
        ]

**Terraform Modification**: 
------------------

#. 次にインスタンスの数を増やしてみます。`hello_tf_instance_count`の値を上書きして再度実行します。

    .. code-block:: bash
        $ export TF_VAR_hello_tf_instance_count=2 
        $ terraform plan
        $ terraform apply -auto-approve


.. note::  ちなみに今回は`-auto-approve`というパラメータを使って途中の実行確認を省略しています。AWS(or GCP or Azure)のインスタンスが二つに増えています。Terraformは環境に差分が生じた際はPlanで差分を検出し、差分のみ実施するため既存のリソースには何の影響も及ぼしません。(GCP/Azureの場合はWebブラウザから確認してください。)

    .. code-block:: bash
        $ aws ec2 describe-instances --query "Reservations[].Instances[].{InstanceId:InstanceId,State:State}"
        [
            {
                "InstanceId": "i-00918d5c9466da418",
                "State": {
                    "Code": 48,
                    "Name": "running"
                },
                "Name": "xxx"
            },
            {
                "InstanceId": "i-0b0aea4b4ab27ef4b",
                "State": {
                    "Code": 16,
                    "Name": "running"
                },
                "Name": "xxx"
            }
        ]

**Destroy Environment**: 
------------------

#. 次に`destroy`で環境をリセットします。

    .. code-block:: bash
        $ terraform destroy 

#. 実行ししばらくするとEC2インスタンスが`terminated`の状態になってることがわかるはずです。

    .. code-block:: bash
        $ aws ec2 describe-instances --query "Reservations[].Instances[].{InstanceId:InstanceId,State:State}"
        [
            {
                "InstanceId": "i-00918d5c9466da418",
                "State": {
                    "Code": 48,
                    "Name": "terminated"
                },
                "Name": "xxx"
            },
            {
                "InstanceId": "i-0b0aea4b4ab27ef4b",
                "State": {
                    "Code": 16,
                    "Name": "terminated"
                },
                "Name": "xxx"
            }
        ]


**Enterprise版の価値**: 
------------------

Applyが実行されると`terraform.tfstate`というファイルが生成されます。このファイルは現在のインフラの状態をJson形式で保持しているものですが、次のPlanのタイミングの差分の検出などで扱われ非常に重要です。例えばチームで作業をする際などはこのステートの共有方法をどうやって運用するかなどの考慮が必要になります。

また、このファイルには各リソースのIDのみならずデータベースやAWS環境のシークレットなど様々な機密性の高いデータが含まれておりステートファイルをセキュアに保つことも運用上重要です。

以降の章ではステートファイルのみならず、OSS版ではTerraformを安全に利用するために考慮する必要がある様々な運用上の課題に対してEnterprise版がどのような機能を提供しているかを一つずつ試してみます。


**参考リンク**
- `State <https://www.terraform.io/docs/state/index.html>`_
- `Backends <https://www.terraform.io/docs/backends/index.html>`_
- `init <https://www.terraform.io/docs/commands/init.html>`_
- `plan <https://www.terraform.io/docs/commands/plan.html>`_
- `apply <https://www.terraform.io/docs/commands/apply.html>`_
- `AWS Provider <https://www.terraform.io/docs/providers/aws/index.html>`_


.. toctree::
   :titlesonly:
   :maxdepth: 2
   :caption: Contents:
   :glob:

   


