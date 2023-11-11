# WEB APP AUTOMATICALLY DEPLOYED TO AWS
## Resumen
En el repositorio actual se aplica el conocimiento sobre CloudFormation en AWS. Consiste en desplegar una solución en varias instancias EC2 definidas como parametros por el usuario

*GitHub URL*: https://github.com/camrojass/CloudFormation_Lab.git

## Indice
1. [Vista de Arquitectura](#id1)
2. [Instalación](#id2)
3. [Autor](#id3)
4. [Bibliografía](#id4)

## Vista de Arquitectura <a name="id1"></a>
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/268a47a7-1c7c-42a5-8f0c-ad504b0a4a28)

En la imagen previa se evidencia los compoentes de la solución
* WebServer Security Group Component
```json
      "WebServerSecurityGroup": {
          "Type": "AWS::EC2::SecurityGroup",
          "Properties": {
              "GroupDescription": "Enable HTTP access via port 80",
              "SecurityGroupIngress": [
                  {
                      "IpProtocol": "tcp",
                      "FromPort": "80",
                      "ToPort": "80",
                      "CidrIp": "0.0.0.0/0"
                  },
                  {
                      "IpProtocol": "tcp",
                      "FromPort": "22",
                      "ToPort": "22",
                      "CidrIp": {
                          "Ref": "SSHLocation"
                      }
                  }
              ],
              "VpcId": {
                  "Ref": "VpcId"
              }
          }
      }
```
* Launch Configuration Component
```json
"MyLaunchConfig": {
        "Type": "AWS::AutoScaling::LaunchConfiguration",
        "Metadata": {},
        "Properties": {
            "ImageId": {
                "Fn::FindInMap": [
                    "AWSRegionArch2AMI",
                    {
                        "Ref": "AWS::Region"
                    },
                    {
                        "Fn::FindInMap": [
                            "AWSInstanceType2Arch",
                            {
                                "Ref": "InstanceType"
                            },
                            "Arch"
                        ]
                    }
                ]
            },
            "SecurityGroups": [
                {
                    "Ref": "WebServerSecurityGroup"
                }
            ],
            "InstanceType": {
                "Ref": "InstanceType"
            },
            "KeyName": {
                "Ref": "KeyName"
            },
            "UserData": {}
        }
    }
```
* Listener Component
```json
      "ALBListener": {
          "Type": "AWS::ElasticLoadBalancingV2::Listener",
          "Properties": {
              "DefaultActions": [
                  {
                      "Type": "forward",
                      "TargetGroupArn": {
                          "Ref": "ALBTargetGroup"
                      }
                  }
              ],
              "LoadBalancerArn": {
                  "Ref": "ApplicationLoadBalancer"
              },
              "Port": "80",
              "Protocol": "HTTP"
          }
      }
```
* LoadBalancer Component
```json
    "ApplicationLoadBalancer": {
        "Type": "AWS::ElasticLoadBalancingV2::LoadBalancer",
        "Properties": {
            "Subnets": {
                "Ref": "Subnets"
            }
        }
    }
```
* AutoScaling Component
```json
    "MyAutoScalingGroup": {
        "Type": "AWS::AutoScaling::AutoScalingGroup",
        "Properties": {
            "TargetGroupARNs": [
                {
                    "Ref": "ALBTargetGroup"
                }
            ],
            "VPCZoneIdentifier": {
                "Ref": "Subnets"
            },
            "LaunchConfigurationName": {
                "Ref": "MyLaunchConfig"
            },
            "MinSize": {
                "Ref": "NumberOfInstances"
            },
            "MaxSize": {
                "Ref": "NumberOfInstances"
            }
        }
    }
```
* TargetGroup Component
```json
    "ALBTargetGroup": {
        "Type": "AWS::ElasticLoadBalancingV2::TargetGroup",
        "Properties": {
            "HealthCheckIntervalSeconds": 10,
            "HealthCheckTimeoutSeconds": 5,
            "HealthyThresholdCount": 2,
            "Port": 80,
            "Protocol": "HTTP",
            "UnhealthyThresholdCount": 5,
            "VpcId": {
                "Ref": "VpcId"
            },
            "TargetGroupAttributes": [
                {
                    "Key": "stickiness.enabled",
                    "Value": "true"
                },
                {
                    "Key": "stickiness.type",
                    "Value": "lb_cookie"
                },
                {
                    "Key": "stickiness.lb_cookie.duration_seconds",
                    "Value": "30"
                }
            ]
        }
}
```

## Instalación <a name="id2"></a>
### Adecuación de [Template.json](Template.json)
1. Se desarrolla la solución.
<details><summary>server.js</summary>
<p>

```js
var express = require('express');
var app     = express();

app.get('/', function(req, res) {
    res.sendfile('index.html');
});

app.listen(80);

console.log('Start Server');
```
</details></p>

<details><summary>index.html</summary>
<p>

```html
<!DOCTYPE html>
<html>
  <head>
    <title> Welcome to Webserver! </title>
    <style>
      html {
        color-scheme: light;
      }

      body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
      }
    </style>
  </head>
  <body>
    <p>
      <br>
      <center> Hello World! </center>
      <br>
    </p>
    <center>
      <iframe width="475" height="250" src="https://reloj-alarma.es/embed/hora/Bogota_CO/Colombia/#theme=1&ampm=0&showdate=1" frameborder="0" allowfullscreen></iframe>
    </center>
    <br>
  </body>
</html>
```
</details></p>

2. Se ajustan las precondiciones para ejecutar la solución.
2.1. Como parte de la metada del componente LauchConfiguration

```json
    "Install": {
        "packages": {
            "yum": {
                "httpd": [],
                "php": [],
                "nodejs": []
              }
        }
    }
```
2.2.Como parte de las propiedades del componente LauchConfiguration
```json
  "Properties": {
      "UserData": {
          "Fn::Base64": {
              "Fn::Join": [
                  "",
                  [
                      "#!/bin/bash -xe\n",
                      "yum update -y aws-cfn-bootstrap\n",
                      "# Install the files and packages from the metadata\n",
                      "/opt/aws/bin/cfn-init -v ",
                      "         --stack ",
                      {
                          "Ref": "AWS::StackName"
                      },
                      "         --resource WebServerInstance ",
                      "         --configsets InstallAndRun ",
                      "         --region ",
                      {
                          "Ref": "AWS::Region"
                      },
                      "\n",
                      "# Signal the status from cfn-init\n",
                      "/opt/aws/bin/cfn-signal -e $? ",
                      "         --stack ",
                      {
                          "Ref": "AWS::StackName"
                      },
                      "         --resource WebServerInstance ",
                      "         --region ",
                      {
                          "Ref": "AWS::Region"
                      },
                      "\n",
      "# Run server.js \n",
      "npm server.js \n",
      "\n"
                  ]
              ]
          }
      }
  }
```
3. Se coloca la solución [server.js](server.js) y [index.html](index.html) como parte del template, como parte de la metada del componente LaunchConfiguration
```json
"files": {
    "server.js": {
        "content": {
            "Fn::Join": [
                "",
                [
                    "\n",
                    "var express = require('express');\n",
                    "var app     = express();\n",
                    "\n",
                    "app.get('/', function(req, res) { \n",
                    "    res.sendfile('index.html');  \n",
                    "}); \n",
                    "\n",
                    "app.listen(80); \n",
                    "\n",
                    "console.log('Start Server'); \n"
                ]
            ]
        },
        "mode": "000600",
        "owner": "apache",
        "group": "apache"
    },
    "index.html": {
        "content": {
            "Fn::Join": [
                "",
                [
                    "<!DOCTYPE html>\n",
                    "<html>\n",
                    "  <head>\n",
                    "    <title> Welcome to Webserver! </title>\n",
                    "      <style>\n",
                    "        html { color-scheme: light; } \n",
                    "        body { width: 35em; margin: 0 auto;  \n",
                    "        font-family: Tahoma, Verdana, Arial, sans-serif; } \n",
                    "      </style>\n",
                    "  </head>\n",
                    "  <body>\n",
                    "    <p>\n",
                    "      <br>\n",
                    "        <center> Hello World! </center>\n",
                    "      <br>\n",
                    "    </p>\n",
                    "      <center>\n",
                    "        <iframe width=\"475\" height=\"250\" src=\"https://reloj-alarma.es/embed/hora/Bogota_CO/Colombia/#theme=1&ampm=0&showdate=1\" frameborder=\"0\" allowfullscreen> </iframe>\n",
                    "      </center>\n",
                    "    <br>\n",
                    "  </body>\n",
                    "</html>\n"
                ]
            ]
        },
        "mode": "000600",
        "owner": "apache",
        "group": "apache"
    }
}
```
### Ejecución de Template

1. Ingresar a AWS, a CloudFormation y crear la pila desde un archivo de plantilla
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/18cb1890-dee7-4696-be92-ccfffd9a2372)

2. Ingresar los parámetros de la aplicación
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/24b3ed0a-4de2-460e-9b85-8601c4406e99)
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/666f2bbc-a98d-439a-81a1-b05975d7157f)

3. Posterior a completar el formulario, ejecutar pila
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/200fb29c-07c6-430d-89d9-776d53d2a7d0)

4. Se valida que los recursos hayan sido creados correctamente.
* Recursos
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/f28294c4-c2ec-4f67-8d4f-fb5511a1fdd2)
* Salidas
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/46ad8abb-59db-499f-80fd-be4b1f3b7995)
* Ejecución salida
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/6926df14-54f2-4cf1-b287-a5e238a8de77)
* Instancias creadas
![image](https://github.com/camrojass/CloudFormation_Lab/assets/100396227/fc3bd12a-985b-4313-a763-31f8e2f448dc)

## Autor <a name="id3"></a> ✒️ 
* **Camilo Alejandro Rojas** - *Trabajo y documentación* - [camrojass](https://github.com/camrojass)

## Bibliografía <a name="id4"></a>
* Guía de usuario CloudFormation. URL: https://docs.aws.amazon.com/es_es/AWSCloudFormation/latest/UserGuide/cfn-whatis-concepts.html#cfn-concepts-templates
* Referencia de tipos de recursos y propiedades de AWS. URL: https://docs.aws.amazon.com/es_es/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html
* Cómo publicar aplicaciones. URL: https://docs.aws.amazon.com/es_es/serverlessrepo/latest/devguide/serverlessrepo-how-to-publish.html
* aws-sam-cli-app-templates. URL: https://github.com/aws/aws-sam-cli-app-templates/tree/master
