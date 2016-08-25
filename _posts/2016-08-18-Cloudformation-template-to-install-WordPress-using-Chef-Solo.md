---
layout: post
title:  "Cloudformation template to install WordPress using Chef-Solo"
date:   2016-08-18 15:07:19
categories: [aws, cloudformation, chef, wordpress]
comments: true
---
This Cloudformation template has created to build a stack from scratch starting from <!--more-->`vpc`, `two public subnets` for elastic loadbalancer (elb), `NAT` instance, `private subnet` for webserver and RDS and uses `autoscaling` for scale-up and scale-down. The Cloudformation template bootstrap chef-solo using cfn-init and install [latest][latest] WordPress.


## Cloudformation Template

{% highlight json %}
{
    "AWSTemplateFormatVersion": "2010-09-09",

    "Description": "CloudFormation template to create VPC and install a WordPress deployment using an Amazon RDS database instance for storage. This AWS CloudFormation bootstrap scripts to install Chef Solo and then Chef Solo is used to install a simple WordPress recipe.",

    "Parameters": {

        "01CIDRVPC": {
            "Description": "Enter the CIDR Range for VPC",
            "Default": "10.44.0.0/16",
            "Type": "String",
            "MinLength": "1",
            "MaxLength": "64",
            "AllowedPattern": "[0-9]+\\..+",
            "ConstraintDescription": "can contain only numeric characters and period."
        },
        
        "02CIDRPublic": {
            "Description": "Enter the CIDR Range for VPC",
            "Default": "10.44.0.0/25",
            "Type": "String",
            "MinLength": "1",
            "MaxLength": "64",
            "AllowedPattern": "[0-9]+\\..+",
            "ConstraintDescription": "can contain only numeric characters and period."
        },
        
        "021CIDRPublic": {
            "Description": "Enter the CIDR Range for VPC",
            "Default": "10.44.0.128/25",
            "Type": "String",
            "MinLength": "1",
            "MaxLength": "64",
            "AllowedPattern": "[0-9]+\\..+",
            "ConstraintDescription": "can contain only numeric characters and period."
        },
        
        "03CIDRPrivate": {
            "Description": "Enter the CIDR Range for VPC",
            "Default": "10.44.1.0/25",
            "Type": "String",
            "MinLength": "1",
            "MaxLength": "64",
            "AllowedPattern": "[0-9]+\\..+",
            "ConstraintDescription": "can contain only numeric characters and period."
        },
        "04CIDRPrivate": {
            "Description": "Enter the CIDR Range for VPC",
            "Default": "10.44.1.128/25",
            "Type": "String",
            "MinLength": "1",
            "MaxLength": "64",
            "AllowedPattern": "[0-9]+\\..+",
            "ConstraintDescription": "can contain only numeric characters and period."
        },

        "KeyPair": {
            "Description": "Name of an existing EC2 KeyPair (find or create here: https://console.aws.amazon.com/ec2/v2/home#KeyPairs: )",
            "Type": "AWS::EC2::KeyPair::KeyName",
            "ConstraintDescription": "must be the name of an existing EC2 KeyPair."
        },

        "04ServerAccess": {
            "Description": "CIDR IP range allowed to login to the NAT instance",
            "Type": "String",
            "MinLength": "9",
            "MaxLength": "18",
            "Default": "0.0.0.0/0",
            "AllowedPattern": "(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})/(\\d{1,2})",
            "ConstraintDescription": "must be a valid CIDR range of the form x.x.x.x/x."
        },
        
        "FrontendType": {
            "Description": "Frontend EC2 instance type",
            "Type": "String",
            "Default": "m1.small",
            "AllowedValues": ["t1.micro", "m1.small", "m1.medium", "m1.large", "m1.xlarge", "m2.xlarge", "m2.2xlarge", "m2.4xlarge", "m3.xlarge", "m3.2xlarge", "c1.medium", "c1.xlarge", "cc1.4xlarge", "cc2.8xlarge", "cg1.4xlarge"],
            "ConstraintDescription": "must be a valid EC2 instance type."
        },

        "GroupSize": {
            "Default": "1",
            "Description": "The default number of EC2 instances for the frontend cluster",
            "Type": "Number"
        },
        
        "MaxSize": {
            "Default": "1",
            "Description": "The maximum number of EC2 instances for the frontend",
            "Type": "Number"
        },

        "DBClass": {
            "Default": "db.t2.micro",
            "Description": "Database instance class",
            "Type": "String",
            "AllowedValues": ["db.t2.micro", "db.t2.small", "db.t2.medium", "db.t2.large", "db.m4.large", "db.m4.xlarge", "db.m4.2xlarge", "db.m4.4xlarge", "db.m4.10xlarge", "db.m3.medium", "db.m3.large", "db.m3.xlarge", "db.m3.2xlarge", "db.r3.large", "db.r3.xlarge", "db.r3.2xlarge", "db.r3.4xlarge", "db.r3.8xlarge", "db.m2.xlarge", "db.m2.2xlarge", "db.m2.4xlarge", "db.m1.medium", "db.m1.large", "db.m1.xlarge", "db.t1.micro"],
            "ConstraintDescription": "must select a valid database instance type."
        },

        "DBName": {
            "Default": "wordpress",
            "Description": "The WordPress database name",
            "Type": "String",
            "MinLength": "1",
            "MaxLength": "64",
            "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
            "ConstraintDescription": "must begin with a letter and contain only alphanumeric characters."
        },

        "DBUser": {
            "Default": "admin",
            "NoEcho": "true",
            "Description": "The WordPress database admin account username",
            "Type": "String",
            "MinLength": "1",
            "MaxLength": "16",
            "AllowedPattern": "[a-zA-Z][a-zA-Z0-9]*",
            "ConstraintDescription": "must begin with a letter and contain only alphanumeric characters."
        },

        "DBPassword": {
            "Default": "password",
            "NoEcho": "true",
            "Description": "The WordPress database admin account password",
            "Type": "String",
            "MinLength": "8",
            "MaxLength": "41",
            "AllowedPattern": "[a-zA-Z0-9]*",
            "ConstraintDescription": "must contain only alphanumeric characters."
        },

        "MultiAZDatabase": {
            "Default": "false",
            "Description": "If true, creates a Multi-AZ deployment of the RDS database",
            "Type": "String",
            "AllowedValues": ["true", "false"],
            "ConstraintDescription": "must be either true or false."
        },
        
        "SSHLocation": {
            "Description": " The IP address range that can be used to SSH to the EC2 instances",
            "Type": "String",
            "MinLength": "9",
            "MaxLength": "18",
            "Default": "0.0.0.0/0",
            "AllowedPattern": "(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})\\.(\\d{1,3})/(\\d{1,2})",
            "ConstraintDescription": "must be a valid IP CIDR range of the form x.x.x.x/x."
        },
        
        "SubnetAZ01": {
            "Description": "Availability Zone of the Subnet",
            "Type": "AWS::EC2::AvailabilityZone::Name"
        },
        
        "SubnetAZ02": {
            "Description": "Availability Zone of the Subnet",
            "Type": "AWS::EC2::AvailabilityZone::Name"
        }
    },

    "Mappings": {
        "NatRegionMap": {
            "us-east-1": {
                "AMI": "ami-184dc970"
            },
            "us-west-1": {
                "AMI": "ami-004b0f60"
            },
            "us-west-2": {
                "AMI": "ami-290f4119"
            },
            "eu-west-1": {
                "AMI": "ami-14913f63"
            },
            "eu-central-1": {
                "AMI": "ami-ae380eb3"
            },
            "sa-east-1": {
                "AMI": "ami-8122969c"
            },
            "ap-southeast-1": {
                "AMI": "ami-6aa38238"
            },
            "ap-southeast-2": {
                "AMI": "ami-893f53b3"
            },
            "ap-northeast-1": {
                "AMI": "ami-27d6e626"
            }
        },
        
        "AWSInstanceType2Arch": {
            "t1.micro": {
                "Arch": "64"
            },
            "m1.small": {
                "Arch": "64"
            },
            "m1.medium": {
                "Arch": "64"
            },
            "m1.large": {
                "Arch": "64"
            },
            "m1.xlarge": {
                "Arch": "64"
            },
            "m2.xlarge": {
                "Arch": "64"
            },
            "m2.2xlarge": {
                "Arch": "64"
            },
            "m2.4xlarge": {
                "Arch": "64"
            },
            "m3.xlarge": {
                "Arch": "64"
            },
            "m3.2xlarge": {
                "Arch": "64"
            },
            "c1.medium": {
                "Arch": "64"
            },
            "c1.xlarge": {
                "Arch": "64"
            },
            "cc1.4xlarge": {
                "Arch": "64HVM"
            },
            "cc2.8xlarge": {
                "Arch": "64HVM"
            },
            "cg1.4xlarge": {
                "Arch": "64HVM"
            }
        },

        "AWSRegionArch2AMI": {
            "us-east-1": {
                "64": "ami-2a69aa47",
                "64HVM": "ami-0da96764"
            },
            "us-west-2": {
                "64": "ami-7f77b31f",
                "64HVM": "NOT_YET_SUPPORTED"
            },
            "us-west-1": {
                "64": "ami-a2490dc2",
                "64HVM": "NOT_YET_SUPPORTED"
            },
            "eu-west-1": {
                "64": "ami-4cdd453f",
                "64HVM": "NOT_YET_SUPPORTED"
            },
            "ap-southeast-1": {
                "64": "ami-df9e4cbc",
                "64HVM": "NOT_YET_SUPPORTED"
            },
            "ap-southeast-2": {
                "64": "ami-63351d00",
                "64HVM": "NOT_YET_SUPPORTED"
            },
            "ap-northeast-1": {
                "64": "ami-3e42b65f",
                "64HVM": "NOT_YET_SUPPORTED"
            },
            "sa-east-1": {
                "64": "ami-1ad34676",
                "64HVM": "NOT_YET_SUPPORTED"
            }
        }
    },

    "Resources": {

        "VPC": {
            "Type": "AWS::EC2::VPC",
            "Properties": {
                "CidrBlock": {
                    "Ref": "01CIDRVPC"
                },
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Public"
                    },
                    {
                        "Key": "Name",
                        "Value": "NAT VPC"
                    }
        ]
            }
        },

        "PublicSubnet01": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::Subnet",
            "Properties": {
                "AvailabilityZone": {
                    "Ref": "SubnetAZ01"
                },
                "VpcId": {
                    "Ref": "VPC"
                },
                "CidrBlock": {
                    "Ref": "02CIDRPublic"
                },
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Public"
                    },
                    {
                        "Key": "Name",
                        "Value": "Public Subnet 01"
                    }
        ]
            }
        },

        "PublicSubnet02": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::Subnet",
            "Properties": {
                "AvailabilityZone": {
                    "Ref": "SubnetAZ02"
                },
                "VpcId": {
                    "Ref": "VPC"
                },
                "CidrBlock": {
                    "Ref": "021CIDRPublic"
                },
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Public"
                    },
                    {
                        "Key": "Name",
                        "Value": "Public Subnet 02"
                    }
                        ]
            }
        },

        "InternetGateway": {
            "Type": "AWS::EC2::InternetGateway",
            "Properties": {
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Public"
                    }
                        ]
            }
        },

        "GatewayToInternet": {
            "DependsOn": ["VPC", "InternetGateway"],
            "Type": "AWS::EC2::VPCGatewayAttachment",
            "Properties": {
                "VpcId": {
                    "Ref": "VPC"
                },
                "InternetGatewayId": {
                    "Ref": "InternetGateway"
                }
            }
        },

        "PublicRouteTable": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::RouteTable",
            "Properties": {
                "VpcId": {
                    "Ref": "VPC"
                },
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Public"
                    }
        ]
            }
        },

        "PublicRoute": {
            "DependsOn": ["PublicRouteTable", "InternetGateway"],
            "Type": "AWS::EC2::Route",
            "Properties": {
                "RouteTableId": {
                    "Ref": "PublicRouteTable"
                },
                "DestinationCidrBlock": "0.0.0.0/0",
                "GatewayId": {
                    "Ref": "InternetGateway"
                }
            }
        },

        "PublicSubnetRouteTableAssociation01": {
            "DependsOn": ["PublicSubnet01", "PublicRouteTable"],
            "Type": "AWS::EC2::SubnetRouteTableAssociation",
            "Properties": {
                "SubnetId": {
                    "Ref": "PublicSubnet01"
                },
                "RouteTableId": {
                    "Ref": "PublicRouteTable"
                }
            }
        },

        "PublicSubnetRouteTableAssociation02": {
            "DependsOn": ["PublicSubnet01", "PublicRouteTable"],
            "Type": "AWS::EC2::SubnetRouteTableAssociation",
            "Properties": {
                "SubnetId": {
                    "Ref": "PublicSubnet02"
                },
                "RouteTableId": {
                    "Ref": "PublicRouteTable"
                }
            }
        },

        "PrivateSubnet01": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::Subnet",
            "Properties": {
                "AvailabilityZone": {
                    "Ref": "SubnetAZ01"
                },
                "VpcId": {
                    "Ref": "VPC"
                },
                "CidrBlock": {
                    "Ref": "03CIDRPrivate"
                },
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Private"
                    },
                    {
                        "Key": "Name",
                        "Value": "Private Subnet 01"
                    }
        ]
            }
        },

        "PrivateSubnet02": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::Subnet",
            "Properties": {
                "AvailabilityZone": {
                    "Ref": "SubnetAZ02"
                },
                "VpcId": {
                    "Ref": "VPC"
                },
                "CidrBlock": {
                    "Ref": "04CIDRPrivate"
                },
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Private"
                    },
                    {
                        "Key": "Name",
                        "Value": "Private Subnet 02"
                    }
        ]
            }
        },

        "PrivateRouteTable": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::RouteTable",
            "Properties": {
                "VpcId": {
                    "Ref": "VPC"
                },
                "Tags": [
                    {
                        "Key": "Application",
                        "Value": {
                            "Ref": "AWS::StackName"
                        }
                    },
                    {
                        "Key": "Network",
                        "Value": "Private"
                    }
        ]
            }
        },

        "PrivateSubnetRouteTableAssociation01": {
            "DependsOn": ["PrivateSubnet01", "PrivateRouteTable"],
            "Type": "AWS::EC2::SubnetRouteTableAssociation",
            "Properties": {
                "SubnetId": {
                    "Ref": "PrivateSubnet01"
                },
                "RouteTableId": {
                    "Ref": "PrivateRouteTable"
                }
            }
        },

        "PrivateSubnetRouteTableAssociation02": {
            "DependsOn": ["PrivateSubnet02", "PrivateRouteTable"],
            "Type": "AWS::EC2::SubnetRouteTableAssociation",
            "Properties": {
                "SubnetId": {
                    "Ref": "PrivateSubnet02"
                },
                "RouteTableId": {
                    "Ref": "PrivateRouteTable"
                }
            }
        },

        "NatSecurityGroup": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::SecurityGroup",
            "Properties": {
                "GroupDescription": "NAT Security Group",
                "VpcId": {
                    "Ref": "VPC"
                },
                "SecurityGroupIngress": [
                    {
                        "IpProtocol": "tcp",
                        "FromPort": "80",
                        "ToPort": "80",
                        "CidrIp": "0.0.0.0/0"
                    },
                    {
                        "IpProtocol": "tcp",
                        "FromPort": "443",
                        "ToPort": "443",
                        "CidrIp": "0.0.0.0/0"
                    },
                    {
                        "IpProtocol": "tcp",
                        "FromPort": "22",
                        "ToPort": "22",
                        "CidrIp": "0.0.0.0/0"
                    }],
                "SecurityGroupEgress": [
                    {
                        "IpProtocol": "tcp",
                        "FromPort": "80",
                        "ToPort": "80",
                        "CidrIp": "0.0.0.0/0"
                    },
                    {
                        "IpProtocol": "tcp",
                        "FromPort": "443",
                        "ToPort": "443",
                        "CidrIp": "0.0.0.0/0"
                    },
                    {
                        "IpProtocol": "tcp",
                        "FromPort": "22",
                        "ToPort": "22",
                        "CidrIp": "0.0.0.0/0"
                    }],
                "Tags": [
                    {
                        "Key": "Name",
                        "Value": "NAT Security Group"
                    }
        ]
            }
        },

        "NatSecurityGroupIngress1": {
            "DependsOn": ["NatSecurityGroup"],
            "Type": "AWS::EC2::SecurityGroupIngress",
            "Properties": {
                "GroupId": {
                    "Ref": "NatSecurityGroup"
                },
                "IpProtocol": "icmp",
                "FromPort": "-1",
                "ToPort": "-1",
                "SourceSecurityGroupId": {
                    "Ref": "NatSecurityGroup"
                }
            }
        },

        "NATIPAddress": {
            "Type": "AWS::EC2::EIP",
            "DependsOn": "GatewayToInternet",
            "Properties": {
                "Domain": "vpc",
                "InstanceId": {
                    "Ref": "NAT"
                }
            }
        },

        "NAT": {
            "DependsOn": ["PublicSubnet01", "PublicSubnet02", "NatSecurityGroup"],
            "Type": "AWS::EC2::Instance",
            "Properties": {
                "InstanceType": "t2.micro",
                "KeyName": {
                    "Ref": "KeyPair"
                },
                "SourceDestCheck": "false",
                "ImageId": {
                    "Fn::FindInMap": ["NatRegionMap", {
                        "Ref": "AWS::Region"
                    }, "AMI"]
                },
                "NetworkInterfaces": [{
                    "GroupSet": [{
                        "Ref": "NatSecurityGroup"
                    }],
                    "AssociatePublicIpAddress": "true",
                    "DeviceIndex": "0",
                    "DeleteOnTermination": "true",
                    "SubnetId": {
                        "Ref": "PublicSubnet01"
                    }
        }],
                "Tags": [
                    {
                        "Key": "Name",
                        "Value": "NAT"
                    }
        ],
                "UserData": {
                    "Fn::Base64": {
                        "Fn::Join": ["", [
	  "#!/bin/bash\n",
	  "yum update -y && yum install -y yum-cron && chkconfig yum-cron on"
	]]
                    }
                }
            }
        },

        "PrivateRoute": {
            "DependsOn": ["PrivateRouteTable", "NAT"],
            "Type": "AWS::EC2::Route",
            "Properties": {
                "RouteTableId": {
                    "Ref": "PrivateRouteTable"
                },
                "DestinationCidrBlock": "0.0.0.0/0",
                "InstanceId": {
                    "Ref": "NAT"
                }
            }
        },

        "ElasticLoadBalancer": {
            "DependsOn": "NAT",
            "Type": "AWS::ElasticLoadBalancing::LoadBalancer",
            "Properties": {
                "SecurityGroups": [{
                    "Ref": "LoadBalancerSecurityGroup"
                }],
                "Subnets": [{
                    "Ref": "PublicSubnet01"
                }, {
                    "Ref": "PublicSubnet02"
                }],
                "Listeners": [
                    {
                        "InstancePort": 80,
                        "Protocol": "HTTP",
                        "LoadBalancerPort": "80"
          }
        ],
                "HealthCheck": {
                    "HealthyThreshold": "2",
                    "Timeout": "5",
                    "Interval": "10",
                    "UnhealthyThreshold": "5",
                    "Target": "HTTP:80/wp-admin/install.php"
                }
            }
        },

        "WebServerGroup": {
            "DependsOn": "NAT",
            "Type": "AWS::AutoScaling::AutoScalingGroup",
            "Properties": {
                "VPCZoneIdentifier": [{
                    "Ref": "PrivateSubnet01"
                }, {
                    "Ref": "PrivateSubnet02"
                }],
                "LoadBalancerNames": [{
                    "Ref": "ElasticLoadBalancer"
                }],
                "LaunchConfigurationName": {
                    "Ref": "LaunchConfig"
                },
                "MinSize": "0",
                "MaxSize": {
                    "Ref": "MaxSize"
                },
                "DesiredCapacity": {
                    "Ref": "GroupSize"
                }
            }
        },

        "LaunchConfig": {
            "DependsOn": ["NAT"],
            "Type": "AWS::AutoScaling::LaunchConfiguration",
            "Metadata": {
                "AWS::CloudFormation::Init": {
                    "configSets": {
                        "ascending": ["remove", "chefversion", "gems"]
                    },
                    "remove": {
                        "commands": {
                            "package-remove-command": {
                                "command": "sudo yum erase ruby* -y "
                            }
                        }
                    },

                    "gems": {
                        "packages": {
                            "rubygems": {
                                "net-ssh": ["3.2.0"],
                                "net-ssh-gateway": ["1.2.0"],
                                "aws-sdk": [],
                                "ffi-yajl": ["2.1"],
                                "chef": ["12.5.1"]
                            }
                        }
                    },


                    "chefversion": {
                        "packages": {
                            "yum": {
                                "mysql56-devel": [],
                                "ruby22*": [],
                                "aws-amitools-ec2": [],
                                "gcc-c++": [],
                                "autoconf": [],
                                "make": [],
                                "automake": [],
                                "libxml2*": [],
                                "libxslt*": []
                            }
                        },

                        "files": {
                            "/etc/chef/solo.rb": {
                                "content": {
                                    "Fn::Join": ["\n", [
                    "log_level :info",
                  	"log_location STDOUT",
                  	"file_cache_path \"/var/chef-solo\"",
                  	"cookbook_path \"/var/chef-solo/cookbooks\"",
                  	"json_attribs \"/etc/chef/node.json\"",
                  	"recipe_url \"https://s3-us-west-2.amazonaws.com/aws-team/491457/wordpress.tar.gz\""
                	]]
                                },
                                "mode": "000644",
                                "owner": "root",
                                "group": "wheel"
                            },
                            "/etc/chef/node.json": {
                                "content": {
                                    "wordpress": {
                                        "db": {
                                            "database": {
                                                "Ref": "DBName"
                                            },
                                            "user": {
                                                "Ref": "DBUser"
                                            },
                                            "host": {
                                                "Fn::GetAtt": ["DBInstance", "Endpoint.Address"]
                                            },
                                            "password": {
                                                "Ref": "DBPassword"
                                            }
                                        }
                                    },
                                    "run_list": ["recipe[wordpress]"]
                                },
                                "mode": "000644",
                                "owner": "root",
                                "group": "wheel"
                            }
                        }
                    }
                }
            },
            "Properties": {
                "InstanceType": {
                    "Ref": "FrontendType"
                },
                "SecurityGroups": [{
                    "Ref": "SSHGroup"
                }, {
                    "Ref": "FrontendGroup"
                }],
                "ImageId": {
                    "Fn::FindInMap": ["AWSRegionArch2AMI", {
                            "Ref": "AWS::Region"
                        },
                        {
                            "Fn::FindInMap": ["AWSInstanceType2Arch", {
                                "Ref": "FrontendType"
                            }, "Arch"]
                        }]
                },
                "KeyName": {
                    "Ref": "KeyPair"
                },
                "UserData": {
                    "Fn::Base64": {
                        "Fn::Join": ["", [
          "#!/bin/bash\n",
          "yum update -y aws-cfn-bootstrap\n",

          "/opt/aws/bin/cfn-init -s ", {
                                "Ref": "AWS::StackId"
                            }, " -r LaunchConfig ", " -c ascending ",
          "         --region ", {
                                "Ref": "AWS::Region"
                            }, " && ",
          "sudo /usr/local/bin/chef-solo -o wordpress::default\n",
          "/opt/aws/bin/cfn-signal -e $? '", {
                                "Ref": "WaitHandle"
                            }, "'\n"
        ]]
                    }
                }
            }
        },

        "WaitHandle": {
            "Type": "AWS::CloudFormation::WaitConditionHandle"
        },

        "WaitCondition": {
            "Type": "AWS::CloudFormation::WaitCondition",
            "DependsOn": "WebServerGroup",
            "Properties": {
                "Handle": {
                    "Ref": "WaitHandle"
                },
                "Timeout": "3600"
            }
        },

        "DBSubnetGroup": {
            "Type": "AWS::RDS::DBSubnetGroup",
            "Properties": {
                "DBSubnetGroupDescription": "Subnets available for the RDS DB Instance",
                "SubnetIds": [
                    {
                        "Ref": "PrivateSubnet01"
                    },
                    {
                        "Ref": "PrivateSubnet02"
                    }
         ]
            }
        },

        "VPCSecurityGroup": {
            "Type": "AWS::EC2::SecurityGroup",
            "Properties": {
                "GroupDescription": "Security group for RDS DB Instance.",
                "VpcId": {
                    "Ref": "VPC"
                },
                "SecurityGroupIngress": [{
                    "IpProtocol": "tcp",
                    "FromPort": "3306",
                    "ToPort": "3306",
                    "CidrIp": {
                        "Ref": "03CIDRPrivate"
                    }
                }, {
                    "IpProtocol": "tcp",
                    "FromPort": "3306",
                    "ToPort": "3306",
                    "CidrIp": {
                        "Ref": "04CIDRPrivate"
                    }
                }]
            }
        },

        "DBInstance": {
            "DependsOn": ["VPCSecurityGroup"],
            "Type": "AWS::RDS::DBInstance",
            "Properties": {
                "DBSubnetGroupName": {
                    "Ref": "DBSubnetGroup"
                },
                "Engine": "MySQL",
                "DBName": {
                    "Ref": "DBName"
                },
                "Port": "3306",
                "MultiAZ": {
                    "Ref": "MultiAZDatabase"
                },
                "MasterUsername": {
                    "Ref": "DBUser"
                },
                "DBInstanceClass": {
                    "Ref": "DBClass"
                },
                "AllocatedStorage": "5",
                "MasterUserPassword": {
                    "Ref": "DBPassword"
                },
                "VPCSecurityGroups": [{
                    "Ref": "VPCSecurityGroup"
                }]
            }
        },

        "LoadBalancerSecurityGroup": {
            "DependsOn": ["VPC"],
            "Type": "AWS::EC2::SecurityGroup",
            "Properties": {
                "GroupDescription": "Enable HTTP & HTTPS",
                "VpcId": {
                    "Ref": "VPC"
                },
                "SecurityGroupIngress": [{
                        "IpProtocol": "tcp",
                        "FromPort": "80",
                        "ToPort": "80",
                        "CidrIp": "0.0.0.0/0"
                    },
                    {
                        "IpProtocol": "tcp",
                        "FromPort": "80",
                        "ToPort": "80",
                        "CidrIp": "0.0.0.0/0"
                    }]
            }
        },

        "SSHGroup": {
            "DependsOn": ["PrivateSubnet01", "PrivateSubnet02"],
            "Type": "AWS::EC2::SecurityGroup",
            "Properties": {
                "VpcId": {
                    "Ref": "VPC"
                },
                "GroupDescription": "Enable SSH access",
                "SecurityGroupIngress": [{
                    "IpProtocol": "tcp",
                    "FromPort": "22",
                    "ToPort": "22",
                    "CidrIp": {
                        "Ref": "SSHLocation"
                    }
                }]
            }
        },

        "FrontendGroup": {
            "DependsOn": ["PrivateSubnet01", "PrivateSubnet02"],
            "Type": "AWS::EC2::SecurityGroup",
            "Properties": {
                "VpcId": {
                    "Ref": "VPC"
                },
                "GroupDescription": "Enable HTTP access via port 80",
                "SecurityGroupIngress": [{
                    "IpProtocol": "tcp",
                    "FromPort": "80",
                    "ToPort": "80",
                    "CidrIp": "0.0.0.0/0"
                }]
            }
        }
    },

    "Outputs": {
        "WebsiteURL": {
            "Value": {
                "Fn::Join": ["", ["http://", {
                    "Fn::GetAtt": ["ElasticLoadBalancer", "DNSName"]
                }, "/"]]
            },
            "Description": "URL to install WordPress"
        },
        "InstallURL": {
            "Value": {
                "Fn::Join": ["", ["http://", {
                    "Fn::GetAtt": ["ElasticLoadBalancer", "DNSName"]
                }, "/wp-admin/install.php"]]
            },
            "Description": "URL to install WordPress"
        },
        "NATIP": {
            "Description": "NAT IP address",
            "Value": {
                "Fn::GetAtt": ["NAT", "PublicIp"]
            }
        }
    }

}
{% endhighlight %}

Download from [Github][githublink]

[latest]:   https://wordpress.org/latest.zip
[githublink]: https://github.com/cloudwaves/cfn-templates.git