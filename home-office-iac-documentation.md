# Infraestrutura como Código para Arquitetura de Home Office na Azure

Esta documentação descreve os scripts de Infraestrutura como Código (IaC) necessários para implantar a arquitetura de solução de home office na plataforma Microsoft Azure.

## Pré-requisitos
- Assinatura ativa da Microsoft Azure
- Instalação do Azure CLI ou Azure PowerShell
- Familiaridade com modelos Azure Resource Manager (ARM)

## Estrutura dos Recursos
A arquitetura é composta pelos seguintes recursos principais no Azure:

1. **Resource Group**: O grupo de recursos que contém todos os recursos da solução.
2. **Virtual Network**: A rede virtual que fornece a conectividade entre os ambientes.
3. **Subnets**:
   - Subnet privada app: Sub-rede para as aplicações acessadas pelos home offices.
   - Subnet privada BD: Sub-rede isolada para os recursos de banco de dados.
4. **VPN Gateway**: O gateway VPN que permite a conexão VPN ponto a site (P2S) dos home offices.
5. **Virtual Machines**: As máquinas virtuais que hospedam os bancos de dados.

## Modelo ARM
Abaixo está o modelo ARM que define todos os recursos necessários para a arquitetura de home office:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "resourceGroupName": {
            "type": "string",
            "metadata": {
                "description": "Nome do Grupo de Recursos"
            }
        },
        "location": {
            "type": "string",
            "defaultValue": "eastus",
            "metadata": {
                "description": "Região do Azure"
            }
        },
        "virtualNetworkName": {
            "type": "string",
            "defaultValue": "vnet-homeoffice",
            "metadata": {
                "description": "Nome da Rede Virtual"
            }
        },
        "appSubnetName": {
            "type": "string",
            "defaultValue": "snet-app",
            "metadata": {
                "description": "Nome da Subnet Privada de Aplicações"
            }
        },
        "dbSubnetName": {
            "type": "string",
            "defaultValue": "snet-db",
            "metadata": {
                "description": "Nome da Subnet Privada de Banco de Dados"
            }
        },
        "vpnGatewayName": {
            "type": "string",
            "defaultValue": "vgw-homeoffice",
            "metadata": {
                "description": "Nome do Gateway VPN"
            }
        },
        "vmCount": {
            "type": "int",
            "defaultValue": 3,
            "metadata": {
                "description": "Número de Máquinas Virtuais de Banco de Dados"
            }
        }
    },
    "resources": [
        {
            "type": "Microsoft.Network/virtualNetworks",
            "name": "[parameters('virtualNetworkName')]",
            "location": "[parameters('location')]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.0.0.0/16"
                    ]
                },
                "subnets": [
                    {
                        "name": "[parameters('appSubnetName')]",
                        "properties": {
                            "addressPrefix": "10.0.1.0/24"
                        }
                    },
                    {
                        "name": "[parameters('dbSubnetName')]",
                        "properties": {
                            "addressPrefix": "10.0.2.0/24"
                        }
                    }
                ]
            }
        },
        {
            "type": "Microsoft.Network/virtualNetworkGateways",
            "name": "[parameters('vpnGatewayName')]",
            "location": "[parameters('location')]",
            "properties": {
                "gatewayType": "Vpn",
                "vpnType": "RouteBased",
                "enableBgp": false,
                "activeActive": false,
                "gatewaySize": "VpnGw1",
                "vpnClientConfiguration": {
                    "vpnClientAddressPool": {
                        "addressPrefixes": [
                            "172.16.201.0/24"
                        ]
                    },
                    "vpnClientProtocols": [
                        "IkeV2"
                    ]
                },
                "ipConfigurations": [
                    {
                        "publicIpAddressName": "pip-vpngw",
                        "publicIpAddressId": "[resourceId('Microsoft.Network/publicIpAddresses', 'pip-vpngw')]",
                        "subnetId": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('virtualNetworkName'), 'GatewaySubnet')]"
                    }
                ]
            },
            "dependsOn": [
                "[resourceId('Microsoft.Network/virtualNetworks', parameters('virtualNetworkName'))]"
            ]
        },
        {
            "type": "Microsoft.Compute/virtualMachines",
            "name": "[concat('db-', copyIndex(1))]",
            "location": "[parameters('location')]",
            "copy": {
                "name": "vmCopy",
                "count": "[parameters('vmCount')]"
            },
            "properties": {
                "hardwareProfile": {
                    "vmSize": "Standard_DS2_v2"
                },
                "osProfile": {
                    "computerName": "[concat('db-', copyIndex(1))]",
                    "adminUsername": "azureuser",
                    "adminPassword": "ComplexPassword123!"
                },
                "storageProfile": {
                    "imageReference": {
                        "publisher": "MicrosoftSQLServer",
                        "offer": "SQL2019-WS2019",
                        "sku": "Standard",
                        "version": "latest"
                    },
                    "osDisk": {
                        "createOption": "FromImage"
                    }
                },
                "networkProfile": {
                    "networkInterfaces": [
                        {
                            "id": "[resourceId('Microsoft.Network/networkInterfaces', concat('nic-db-', copyIndex(1)))]"
                        }
                    ]
                }
            },
            "dependsOn": [
                "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('virtualNetworkName'), parameters('dbSubnetName'))]",
                "[resourceId('Microsoft.Network/networkInterfaces', concat('nic-db-', copyIndex(1)))]"
            ]
        },
        {
            "type": "Microsoft.Network/networkInterfaces",
            "name": "[concat('nic-db-', copyIndex(1))]",
            "location": "[parameters('location')]",
            "copy": {
                "name": "nicCopy",
                "count": "[parameters('vmCount')]"
            },
            "properties": {
                "ipConfigurations": [
                    {
                        "name": "ipconfig1",
                        "properties": {
                            "subnet": {
                                "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', parameters('virtualNetworkName'), parameters('dbSubnetName'))]"
                            },
                            "privateIPAllocationMethod": "Dynamic"
                        }
                    }
                ]
            },
            "dependsOn": [
                "[resourceId('Microsoft.Network/virtualNetworks', parameters('virtualNetworkName'))]"
            ]
        }
    ],
    "outputs": {
        "virtualNetworkId": {
            "type": "string",
            "value": "[resourceId('Microsoft.Network/virtualNetworks', parameters('virtualNetworkName'))]"
        },
        "vpnGatewayId": {
            "type": "string",
            "value": "[resourceId('Microsoft.Network/virtualNetworkGateways', parameters('vpnGatewayName'))]"
        }
    }
}
```

## Instruções de Implantação
1. Abra o Azure CLI ou Azure PowerShell.
2. Efetue login na sua conta do Azure: `az login` ou `Connect-AzAccount`.
3. Crie um novo grupo de recursos ou use um existente: `az group create -n <resourceGroupName> -l <location>`.
4. Implante o modelo ARM:
   - CLI: `az deployment group create -g <resourceGroupName> --template-file <path/to/template.json>`.
   - PowerShell: `New-AzResourceGroupDeployment -ResourceGroupName <resourceGroupName> -TemplateFile <path/to/template.json>`.
5. Aguarde a conclusão da implantação.
6. Acesse os recursos implantados no portal do Azure.

## Próximos Passos
- Configurar o VPN Gateway para conexão dos home offices.
- Implantar as aplicações na subnet privada de aplicações.
- Configurar os recursos de banco de dados nas máquinas virtuais.
- Testar a conectividade e o acesso dos home offices ao ambiente.

