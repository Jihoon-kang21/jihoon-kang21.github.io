---
layout: post
title: 설치된 python library들의 license, version 정보를 포함한 SBOM 만들기 
subtitle: sbom-utility를 이용하여 cycloneDX 표준을 바탕으로 SBOM을 만들어본다.
categories: tech
tags: [sbom,cyclonedx,python,script]
---

sbom-utility 를 이용하면 설치된 라이브러리들의 버전과 라이선스를 확인할 수 있다. 솔루션을 제공할때 고객사 등에서 관련 자료를 요구할 때 용이하다.
SBOM 은 소프트웨어 설치목록을 의미하고 SBOM을 제공할 때 표준 중 하나가 CycloneDX이다.

사이클론DX는 보안 커뮤니티의 오랜 리더인 국제 웹 보안 표준기구(Open Web Application Security Project, OWASP)가 주도한다. 사이클론DX는 자체적으로 “애플리케이션 보안 컨텍스트 및 공급망 구성 요소 분석에 사용하도록 설계된 경량의 SBOM 표준”이라고 정의한다.

아래에서는 간단하게 sbom-utility를 사용해보기 위해 python 3.8.9 버전 docker image 를 활용한다.

Note : python library 대상으로 실행했을 때는 정보가 없는 library들이 종종 있는데, Go일때는 잘 된다고한다.

### 1. sbom-utiliy 설치

아래 링크에서 다운로드 받는다.
- [https://github.com/CycloneDX/sbom-utility/releases](https://github.com/CycloneDX/sbom-utility/releases)
압축을 풀면 설치는 끝난다.
설치하면 아래와 같은 파일들이 있다.

```
LICENSE                       
config.json                   
license.json                  
sbom-utility-v0.13.0.bom.json
README.md                     
custom.json                   
sbom-utility
```
그중 sbom-utility가 실행파일이다.

### 2. python 환경 준비

- 위 파일들을 sbom이라는 폴더를 만들어 이동해 놓는다.
- sbom 폴더 상위에 docker-compose.yml 파일을 만들고 내용을 다음과 같이 쓴다.

```
version: '3.9'

services:
  python:
    image: python:3.8.9
    container_name: python-3.8.9
    volumes:
      - ./sbom:/sbom
    command:
      - /bin/sh
      - -c
      - |
          sleep 300000
```

- 아래 명령으로 python image에 진입하면 image 내부에서 sbom-utility를 테스트할수 있다.

```bash
docker exec -it python-3.8.9 bash
cd /sbom
```

### 3. library 설치 및 cyclonedx.json 생성

- 먼저 cyclonedx-bom 설치가 필요하다. 
- 그 외 리스트에 포함하고 싶은 library 등을 설치해 본다.
- cyclonedx_py 명령어(CycloneDX SBOM Generator) 로 cyclonedx.json을 생성한다.
	- 포멧은 json 또는 xml로 가능하다.

```bash
#!/bin/bash

pip install cyclonedx-bom

# install all dependencies
pip install -r ./requirements.txt

# Generate report
python -m cyclonedx_py -F -e --format=json
```

아래는 cyclonedx_py 사용법이다.
```
$ cyclonedx-py --help
usage: cyclonedx-py [-h] (-c | -cj | -e | -p | -pip | -r) [-i FILE_PATH]
                 [--format {json,xml}] [--schema-version {1.4,1.3,1.2,1.1,1.0}]
                 [-o FILE_PATH] [-F] [-X]

CycloneDX SBOM Generator

optional arguments:
  -h, --help            show this help message and exit
  -c, --conda           Build a SBOM based on the output from `conda list
                        --explicit` or `conda list --explicit --md5`
  -cj, --conda-json     Build a SBOM based on the output from `conda list
                        --json`
  -e, --e, --environment
                        Build a SBOM based on the packages installed in your
                        current Python environment (default)
  -p, --p, --poetry     Build a SBOM based on a Poetry poetry.lock's contents.
                        Use with -i to specify absolute path to a `poetry.lock`
                        you wish to use, else we'll look for one in the
                        current working directory.
  -pip, --pip           Build a SBOM based on a PipEnv Pipfile.lock's
                        contents. Use with -i to specify absolute path to a
                        `Pipefile.lock` you wish to use, else we'll look for
                        one in the current working directory.
  -r, --r, --requirements
                        Build a SBOM based on a requirements.txt's contents.
                        Use with -i to specify absolute path to a
                        `requirements.txt` you wish to use, else we'll look
                        for one in the current working directory.
  -X                    Enable debug output

Input Method:
  Flags to determine how this tool obtains it's input

  -i FILE_PATH, --in-file FILE_PATH
                        File to read input from, or STDIN if not specified

SBOM Output Configuration:
  Choose the output format and schema version

  --format {json,xml}   The output format for your SBOM (default: xml)
  --schema-version {1.4,1.3,1.2,1.1,1.0}
                        The CycloneDX schema version for your SBOM (default:
                        1.3)
  -o FILE_PATH, --o FILE_PATH, --output FILE_PATH
                        Output file path for your SBOM (set to '-' to output
                        to STDOUT)
  -F, --force           If outputting to a file and the stated file already
                        exists, it will be overwritten.
```

### 4. 설치된 라이브러리들의 라이선스 확인하기

- license list 명령어로 라이선스 정보를 포함한 목록을 만든다.
- query 를 이용하며 필요한 author, version, license 정보를 가져온다.

```bash
./sbom-utility license list -i cyclonedx.json --summary --quiet --format csv > license_report.csv

./sbom-utility query -i ./cyclonedx.json --quiet --select "type,name,author,version,licenses" --from components --where "name=.*" > license_dependencies.json
```

- license 명령어 결과에 대한 설명이다.
	- **Notes**
	    - **Usage policy** column values are derived from the `license.json` policy configuration file.
	        - A `usage policy` value of `UNDEFINED` indicates that `license.json` provided no entry that matched the declared license (`id` or `name`) in the SBOM.
	    - **License expressions** (e.g., `(MIT or GPL-2.0)`) with one term resolving to `UNDEFINED` and the the other term having a concrete policy will resolve to the "optimistic" policy for `OR` expressions and the "pessimistic" policy for `AND` expressions. In addition, a warning of this resolution is emitted.

- license 명령어로 만들어진 결과물 예시
```bash
$ cat license_report.csv
usage-policy,license-type,license,resource-name,bom-ref,bom-location
allow,name,Apache 2.0,sortedcontainers,698a8e4d-bfe6-4c1d-942f-a89bbd5b8978,components
allow,name,Apache Software License,cyclonedx-bom,7f05d4e1-27b5-451e-ab69-a4105f804d53,components
allow,name,Apache Software License,cyclonedx-python-lib,3e9f1380-acba-4478-84c1-bb21cdebd8c1,components
allow,name,Apache Software License,packaging,5ea51ba8-b735-4c74-a38b-e0cca556f87a,components
allow,name,Apache Software License,sortedcontainers,698a8e4d-bfe6-4c1d-942f-a89bbd5b8978,components
allow,name,Apache-2.0,cyclonedx-bom,7f05d4e1-27b5-451e-ab69-a4105f804d53,components
allow,name,Apache-2.0,cyclonedx-python-lib,3e9f1380-acba-4478-84c1-bb21cdebd8c1,components
allow,name,BSD License,packaging,5ea51ba8-b735-4c74-a38b-e0cca556f87a,components
needs-review,name,GNU Lesser General Public License v2 or later (LGPLv2+),chardet,b528ee38-2813-4b15-a946-03940877404b,components
needs-review,name,LGPL,chardet,b528ee38-2813-4b15-a946-03940877404b,components
allow,name,MIT,packageurl-python,f3edb21b-6c75-4826-b70c-b8cf819ec7f1,components
allow,name,MIT,pip,311d0a28-4075-4c73-b0d7-75ce65b098ac,components
allow,name,MIT,pip-requirements-parser,4197d1e4-53ab-4037-a275-ae6b6ebad75f,components
allow,name,MIT,toml,739191dc-ef61-4122-8078-59330157e403,components
allow,name,MIT,wheel,773857da-8b85-46b4-b24f-c976865305af,components
allow,name,MIT License,packageurl-python,f3edb21b-6c75-4826-b70c-b8cf819ec7f1,components
allow,name,MIT License,pip,311d0a28-4075-4c73-b0d7-75ce65b098ac,components
allow,name,MIT License,pyparsing,fc307784-0e5a-42be-945c-7bb6cdea400c,components
allow,name,MIT License,setuptools,3f7c5f34-6828-4752-8577-da5527ea61ab,components
allow,name,MIT License,toml,739191dc-ef61-4122-8078-59330157e403,components
allow,name,MIT License,wheel,773857da-8b85-46b4-b24f-c976865305af,components
UNDEFINED,invalid,NOASSERTION,,,metadata.component
```

- query 명령어 결과 예시 : "type,name,author,version,licenses" 를 select 했을때
```bash
root@deb1249312d7:/app/sbom# ./sbom-utility query -i ./cyclonedx.json --quiet --select "type,name,author,version,licenses" --from components --where "name=cyclonedx-python-lib"
[
  {
    "author": "Paul Horton",
    "licenses": [
      {
        "license": {
          "name": "Apache Software License"
        }
      },
      {
        "license": {
          "name": "Apache-2.0"
        }
      }
    ],
    "name": "cyclonedx-python-lib",
    "type": "library",
    "version": "3.1.5"
  }
]
```

- query 명령어 결과 예시 : 모든 정보를 select 했을때
```bash
root@deb1249312d7:/app/sbom# ./sbom-utility query -i ./cyclonedx.json --quiet --select '*' --from components --where "name=cyclonedx-python-lib"
[
  {
    "author": "Paul Horton",
    "bom-ref": "3e9f1380-acba-4478-84c1-bb21cdebd8c1",
    "licenses": [
      {
        "license": {
          "name": "Apache Software License"
        }
      },
      {
        "license": {
          "name": "Apache-2.0"
        }
      }
    ],
    "name": "cyclonedx-python-lib",
    "purl": "pkg:pypi/cyclonedx-python-lib@3.1.5",
    "type": "library",
    "version": "3.1.5"
  }
]
```

아래는 전체 query 결과이다.
```bash
$ cat license_dependencies.json
[
  {
    "author": "Mark Pilgrim",
    "licenses": [
      {
        "license": {
          "name": "GNU Lesser General Public License v2 or later (LGPLv2+)"
        }
      },
      {
        "license": {
          "name": "LGPL"
        }
      }
    ],
    "name": "chardet",
    "type": "library",
    "version": "5.2.0"
  },
  {
    "author": "Steven Springett",
    "licenses": [
      {
        "license": {
          "name": "Apache Software License"
        }
      },
      {
        "license": {
          "name": "Apache-2.0"
        }
      }
    ],
    "name": "cyclonedx-bom",
    "type": "library",
    "version": "3.11.7"
  },
  {
    "author": "Paul Horton",
    "licenses": [
      {
        "license": {
          "name": "Apache Software License"
        }
      },
      {
        "license": {
          "name": "Apache-2.0"
        }
      }
    ],
    "name": "cyclonedx-python-lib",
    "type": "library",
    "version": "3.1.5"
  },
  {
    "author": "the purl authors",
    "licenses": [
      {
        "license": {
          "name": "MIT"
        }
      },
      {
        "license": {
          "name": "MIT License"
        }
      }
    ],
    "name": "packageurl-python",
    "type": "library",
    "version": "0.11.2"
  },
  {
    "author": null,
    "licenses": [
      {
        "license": {
          "name": "Apache Software License"
        }
      },
      {
        "license": {
          "name": "BSD License"
        }
      }
    ],
    "name": "packaging",
    "type": "library",
    "version": "23.2"
  },
  {
    "author": "The pip developers",
    "licenses": [
      {
        "license": {
          "name": "MIT"
        }
      },
      {
        "license": {
          "name": "MIT License"
        }
      }
    ],
    "name": "pip",
    "type": "library",
    "version": "23.3.1"
  },
  {
    "author": "The pip authors, nexB. Inc. and others",
    "licenses": [
      {
        "license": {
          "name": "MIT"
        }
      }
    ],
    "name": "pip-requirements-parser",
    "type": "library",
    "version": "32.0.1"
  },
  {
    "author": null,
    "licenses": [
      {
        "license": {
          "name": "MIT License"
        }
      }
    ],
    "name": "pyparsing",
    "type": "library",
    "version": "3.1.1"
  },
  {
    "author": "Python Packaging Authority",
    "licenses": [
      {
        "license": {
          "name": "MIT License"
        }
      }
    ],
    "name": "setuptools",
    "type": "library",
    "version": "56.0.0"
  },
  {
    "author": "Grant Jenks",
    "licenses": [
      {
        "license": {
          "name": "Apache 2.0"
        }
      },
      {
        "license": {
          "name": "Apache Software License"
        }
      }
    ],
    "name": "sortedcontainers",
    "type": "library",
    "version": "2.4.0"
  },
  {
    "author": "William Pearson",
    "licenses": [
      {
        "license": {
          "name": "MIT"
        }
      },
      {
        "license": {
          "name": "MIT License"
        }
      }
    ],
    "name": "toml",
    "type": "library",
    "version": "0.10.2"
  },
  {
    "author": "Daniel Holth",
    "licenses": [
      {
        "license": {
          "name": "MIT"
        }
      },
      {
        "license": {
          "name": "MIT License"
        }
      }
    ],
    "name": "wheel",
    "type": "library",
    "version": "0.36.2"
  }
]
```

