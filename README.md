WAF (ModSecurity) WASM Filter as Envoy extension (Istio control plane)
===========
> **Note**: This project is **experimental** and under development.

WAF WASM Filter is a WASM based Envoy filter, developed to be deployed on Istio. It is based on [Libmodsecurity](https://github.com/SpiderLabs/ModSecurity) (ModSecurity v3), the C++ library of the common open source Web Application Firewall, and on [WebAssembly for Proxies (C++ SDK)](https://github.com/proxy-wasm/proxy-wasm-cpp-sdk)).
WAF functionalities, implemented as a WebAssembly module, extend the Envoy proxy security capabilities across the Istio service mesh. In terms of detection, the WAF relies on most of the rules provided by the [OWASP CoreRuleSet](https://owasp.org/www-project-modsecurity-core-rule-set/) (**CRS**) v.3.3.2.

> **Note**: Check the Current Limitations section to see details about the excluded rules.

## Repository Structure

- **/basemodsec**: Main project folder.
- **/demo**: Basic examples of YAML files to deploy the WAF filter.
- **/modsec_rules_collection/**:
    - `/coreruleset-3.3.2-rules`: Original collection of rules from CRS v.3.3.2.
    - `excluded rules.txt`: File listing the excluded rules.
    - `raw_wip_rules_collection.txt`: Raw collection of working custom rules.
    - `rulethemall_orig.conf`: Concatenation of all the CRS rules hardecoded inside the application.
    - `rulethemall.conf`: Stripped version of rulethemall_orig.conf.
    - `sqlirules.conf`: Stripped version of the CRS `REQUEST-941-APPLICATION-ATTACK-XSS.conf` file.
    - `xssrules.conf`: Stripped version of the CRS `REQUEST-942-APPLICATION-ATTACK-SQLI.conf` file.
- **/wasm**: Collection of compiled wasm files. `_nodebug` suffix means that the wasm has been compiled with less logs verbosity (e.g. print the whole body of each request). For details navigate the code looking for [DEBUG](/basemodsec/plugin.cc#L24) usage.
- **/yaml**: Collection of yamls examples.


# Quick Deployment Guide

**Prerequisites**:
 - Istio service mesh up and running. See the [official istio.io guide](https://istio.io/latest/docs/setup/getting-started/).
 - (optional) [Istio sample application](https://istio.io/latest/docs/setup/getting-started/#bookinfo) deployed. This guide is based on bookinfo sample environment.

The fastest way to have this project up and running relies on the already built `.wasm` file provided in this repo [here](/wasm/).

It is just needed to:
1. Download the filter deployment example `.yaml` file.
2. (optional) Customize the location of the deployment (default configuration will deploy it inside the istio-proxy of the `productpage` workload)
3. (optional) Customize the Modsecurity rules provided to the WAF (default configuration enables the CRS).
4. Apply the `.yaml` file via `sudo kubectl apply -f file_name.yaml` 
**TODO**: write wasm file name on 1. and 4.

Check the correct deployment:
- Send a request that matches a modsec rule e.g: ```curl -I http://istio.k3s/productpage?arg=<script>alert(0)</script>```. The expected return code is `403`.
> **Note**: the url that has to be contact will depend on how the service has been exposed to external traffic
- Check the sidecar's logs: ```kubectl get logs name_of_the_pod -c istio-proxy```

## Modsecurity Configuration
One key element of this project is to provide enough flexiblity in terms of Modsecurity configuration without the necessity of recompiling each time the whole WASM file. This is achieved via the possibility of providing a JSON string inside the YAML file that is consumed by the Wasm extension.
The current JSON schema expected by the WASM filter is the following one:
```
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "WASM Modsec configuration via YAML",
  "type": "object",
  "properties": {
    "modsec_config": {
      "type": "array",
      "items": [
        {
          "type": "object",
          "properties": {
            "enable_default": {
              "type": "string",
              "enum": [
                "yes",
                "no"
              ],
              "default": "yes"
            },
            "enable_crs": {
              "type": "string",
              "enum": [
                "yes",
                "no"
              ],
              "default": "yes"
            },
            "enable_sqli": {
              "type": "string",
              "enum": [
                "yes",
                "no"
              ],
              "default": "no"
            },
            "enable_xss": {
              "type": "string",
              "enum": [
                "yes",
                "no"
              ],
              "default": "no"
            },
            "custom_rules": {
              "type": "array",
              "items": [
                {
                  "type": "string"
                }
              ]
            }
          },
          "required": []
        }
      ]
    }
  }
}
```
Example of a validated json:
```
{
"modsec_config": [
    {
    "enable_default": "yes",
    "enable_crs": "yes",
    "custom_rules":[
    "SecRule ARGS \"@rx matteo\" \"id:103,phase:1,t:lowercase,deny\"",
    "SecRuleRemoveById 920280"
    ]
    }
]
}
```
Notes about the json configuration:
- `enable_` fields refer to already hardcoded rules inside the application: `enable_default` includes mosts of the basic needed rules coming from [modsecurity.conf](https://github.com/SpiderLabs/ModSecurity/blob/v3/master/modsecurity.conf-recommended) and crs-setup.conf. `enable_crs` enables the almost complete collection of CRS rules. Refer to [rules.cc](TODO) to see the complete list of rules and to [feature requests](TODO) for the current rules limitation.  
No fields are mandatory: default values, as indicated inside the schema, are:
    - `enable_default`: `yes` 
    - `enable_crs`: `yes`
    - `enable_xss`: `no`
    - `enable_sqli`: `no`
- `enable_crs` logically includes `enable_sqli` and `enable_xss`. Enabling it leads the filter to do not take into account any possible values of `enable_sqli` and `enable_xss`.
- For a complete custom configuration it is possible to set `enable_default` and `enable_crs` to `no` and provide all the rules via `custom_rules`.

 

# Developer Guide
This section is inteded to provide details about building from scratch all the components used inside this work and give technical details to maintain, customize and improve the WAF WASM project.

## Building Libmodsecurity for WASM
The project relies on the Libmodsecurity library that has to be ported to WASM architecture. The tool used is [Emscripten](https://emscripten.org/), a compiler toolchain to WebAssembly that permits the porting of C/C++ projects to WASM.

1. Download and install Emscripten:
```
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk
git pull
./emsdk install 2.0.7
./emsdk activate 2.0.7
source ./emsdk_env.sh
```
> **Note**: It is strongly suggested to maintain the same Emscripten version for both libraries and filter building. At the moment of writing, the WASM Filter building process relies on \textbf{Emscripten v 2.07}. Stick with this version or be aware that runtime errors may arise.

2. Download and install the WASI SDK:
```
mkdir /opt/wasi-sdk
cd /opt/wasi-sdk
wget https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-
sdk-12/wasi-sdk-12.0-linux.tar.gz
tar -xvf wasi-sdk-12.0-linux.tar.gz
export WASI_SDK_PATH="/opt/wasi-sdk-12.0"
```

3. Download some dependecies:
```
apt install libtool-bin automake texinfo
```
4. Download the PCRE library specifing the *wasm-wasi branch* and install it via the provided script (`build_for_crystal.sh`).

5. PCRE library is now built as a static library under `./targets/`. Move it inside the project folder under `/pcre/lib/`.
```
cp ./targets/*.a ./basemodsec/pcre/lib
```

6. Download Libmodsecurity customized for WASM build and add the `pcre.h` header file.
```
git clone https://github.com/leyao-daily/ModSecurity.git
cd ModSecurity
cp /usr/include/pcre.h ./headers/
```

7. Built it with minimal dependencies:
```
./build.sh
git submodule init
git submodule update
emconfigure ./configure --without-yajl --without-geoip --without-libxml \
--without-curl --without-lua --disable-shared --disable-examples \
--disable-libtool-lock --disable-debug-logs  --disable-mutex-on-pm\
 --without-lmdb --without-maxmind --without-ssdeep --with-pcre=./pcre-config
emmake make
emmake install
```

8. Libmodsecurity built for WebAssembly architecture is now under `/usr/local/modsecurity/lib`. Copy it and relative header filers inside the project.
```
cp /usr/local/modsecurity/lib/* ./basemodsec/modsec/lib
```
The WASM module is now ready to be built relying on the libraries just compiled and added to the project's folder. The latter should now have the following directory structure:
```
basemodsec
├── modec
│   ├── include
│   └── lib
├── pcre
│   ├── include
│   └── lib
└── ...
```

**TODO**: be clear about the fact that *.a files are not inside this repo and are needed to build it. 

## Building the Filter
### Environment setup
The building process is based on Bazel. The *WebAssembly for Proxies C++ SDK* has dependencies on specific tools versions. Relying on Bazel, and on already available build rules files, permits to hide most of the complexity related to install the correct toolchain. This [link](https://github.com/proxy-wasm/proxy-wasm-cpp-sdk/blob/fd0be8405db25de0264bdb78fae3a82668c03782/bazel/dep/deps.bzl\#L17) shows the build function internally called. Rely on this file to check the version used in case you wish to perform manual builds. At the moment of writing **Emscripten v.2.0.7** and **Protobuf v.3.17.3** are used. If needed, the [GitHub page of the SDK](https://github.com/proxy-wasm/proxy-wasm-cpp-sdk) provides further building details.

1. Bazel can be downloaded via its wrapper Bazelisk.
 ```
sudo wget -O /usr/local/bin/bazel https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-amd64

sudo chmod +x /usr/local/bin/bazel
 ```
2. Install dependencies:
 ```
sudo apt-get install gcc curl python3
 ``` 
 For further details refer to [Istio Wasm Extensions Development Guides](https://github.com/istio-ecosystem/wasm-extensions#development-guides) and its [Set up Develop Environment](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/development-setup.md#set-up-develop-environment).

### Building commands
Go to the main folder of the project and run `bazel build` as follows:
> **Note**: Do **not** perform `bazel build` command as **root** user
```
cd ./basemodsec
bazel build //:basemodsec.wasm
```
The wasm file is now generated under `./bazel-bin/` folder.

For further details refer to [Develop a Wasm extension with C++](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/write-a-wasm-extension-with-cpp.md).

## Deployment
The generated wasm file now has to be deployed.
Two `EnvoyFilter` resources are needed to deploy the just built wasm extension with Istio across the service mesh:
- The first declares the filter as HTTP_FILTER and specifies its position inside the filter chain of envoy.
- The second ones provides configuration to the filter including:
    - how to retrieve the `.wasm` file. Local and remote ways can be used to provide the extension. All yaml files in this repository realies on downloading it from a remote http uri. To further details refer to [Istio documentation](https://istio.io/latest/docs/ops/configuration/extensibility/wasm-module-distribution/).
    - JSON configuration that will be internally handled by the filter at the booting phase.

1. Upload the `.wasm` file to be publicly eccessible from a https request (e.g. inside a GitHub repository).
2. Retrieve a link to directly download the `.wasm` file. e.g. `https://github.com/M4tteoP/wasm-repo/raw/main/basemodsec.wasm`.
3. Customize the deployment according to your needs.
    - specify the namespace and/or the specific workload where the WAF must be deployed.
    - update the download uri.
    - update custom rules and flags that will configure Modsecurity (for details see Modsecurity Configuration TODO add link).
4. Apply the yaml file inside the cluster via `kubectl apply -f file_name.yaml`.

## Implementation Details
The main source code files are:
* `plugin.cc` and `plugin.h` with the whole logic of the application.
* `rules.cc` and `rules.h` with the basic configuration rules and CRS rules hardcoded.

About the application logic, to implement a WASM module, the following elements are mandatory:
* **Implementation of the root context class**: Named `PluginRootContext`, which inherits the `RootContext` class defined inside the SDK. The root context object is created during the bootstrap of the WASM module and has the lifetime of the VM on which the module is executed. Here reside all the elements that have to be kept alive across requests. This is the case of three core elements:
  * `modsecurity::ModSecurity *modsec`: `modsec` points to a `ModSecurity` object, allocated during the initial configuration of the WASM module. It is the main ModSecurity element that, within the following `rules` object, will be used to initialize a transaction and perform security controls every needed time.
  * `modsecurity::RulesSet *rules`: It is another object from libmodsecurity. It points to a `RulesSet` object that gets populated with the rules that will be used by the transaction.
  * `PluginRootContext::ModSecConfigStruct modSecConfig`: It is a custom struct used to keep all configuration elements received from the configuration file. It is made of boolean variables about enabling or not the default configurations, the CRS and specific parts of it (for SQLi and XSS detection). It also includes a vector of strings for the custom rules. Its declaration is:
    ```
    struct ModSecConfigStruct {
      bool enable_default;
      bool enable_crs;
      bool detect_sqli;
      bool detect_xss;
      std::vector<std::string> custom_rules;
    };
    ```
* **Implementation of the stream context class**: Named `PluginContext`, which inherits the `Context` class defined inside the SDK. A context object is created for each steam and is deallocated once the stream itself ends. It is therefore possible to rely on this object just between events of the same stream. The key element that requires to have a visibility of the whole stream is the `Transaction` object of libmodsecurity. It is named `modsecTransaction` and is allocated when a stream begins, as soon as the request headers are handled by the module. As the following snippet of code shows, its instantiation relies on the two elements previously explained in the root context:
  ```
  modsecTransaction = new modsecurity::Transaction(rootContext()->modsec, rootContext()->rules, NULL);
  ```
* **Override context API methods to handle events**: Inside the `PluginContext`, API methods are also defined. They correspond to the callbacks for stream events. The full list of overridden functions for this project is the following:
  * `FilterHeadersStatus onRequestHeaders(uint32_t, bool) override`: Here the transaction object is created and, via the function `initTransaction`, is populated with basic information of the connection (client IP, client port, destination IP, destination port) and of the request (url and method). Once `modsecTransaction` is populated `process_intervention` is applied to analyze the content. Because most of the CRS rules are not applied at phase 1, a little trick is coded: an empty body is added to the transaction. Doing so, the transaction enters phase 2 and the rules are correctly applied.
  * `FilterDataStatus onRequestBody(unsigned long, bool) override`: `onRequestBody` reads the body of the request and adds it to `modsecTransaction`. Afterwards, the process function on the request body is applied (function `processRequestBody`).
  * `FilterHeadersStatus onResponseHeaders(uint32_t, bool) override`: same as the previous `onRequestHeaders`, but applied on the headers of the response.
  * `FilterDataStatus onResponseBody(unsigned long, bool) override`: same as  the previous `onRequestBody`, but applied on the headers of the response.
  * `void onDelete() override`: the `onDelete` callback is triggered when the stream is ended and the stream context is up for deconstruction. Here the `Transaction` object pointed by `modsecTransaction` is deallocated. Skipping this action leads to leak the memory pointed and to a fatal state of the WASM filter.
  

  The `PluginRootContext` includes the API methods for initialization events. Here, the overridden function is just one:
  * `bool onConfigure(size_t) override`: executed just at the startup, after the creation of the root context. It instantiate the `modsec` and `rules` object, parse the JSON object received from the configuration file and populate accordingly both the `{modSecConfig` structure and the `rules` object.

* **Register the root context and stream context**: It is done at the beginning of the `plugin.cc` file with the following static function:
  ```
  static RegisterContextFactory register_Example(CONTEXT_FACTORY(PluginContext), ROOT_FACTORY(PluginRootContext))
  ```
  
Two other functions that worth to be mentioned are `alertActionHeader` and `alertActionBody`. Each `process_intervention` executed inside the callbacks returns a number that, if it is different from `0`, means that a rule has been matched.  Implementing the rules in the traditional mode, one rule is enought to stop the chain and return an error code to the client. It is accomplished by  these functions, that, with minimal differences between them, are as the following one:
```
FilterHeadersStatus PluginContext::alertActionHeader(int response){
    sendLocalResponse(403, absl::StrCat("Dropped by alertActionHeader response= ",std::to_string(response)), "", {});
    return FilterHeadersStatus::StopIteration;
}
```
The `sendLocalResponse` function, defined inside the C++ SDK, permits to send a response back to the client. `403` explicits the return code. Small differences happen based on when the `alertAction` is triggered: stopping the stream at phases 1, 2 or 3 leads to the expected behaviour, with a 403 error page. But, due to the no buffering phases, if a rule is matched at phase 4, the response header is already sent to the client. Therefore, only the body response will be blocked. The client will receive a status `200` with an empty body.

## Debugging Tips
- **Change Envoy log level**: by default, Istio injects the istio-proxy (Envoy) with log levels set as `info`. `trace` and `debug` are more verbose alternatives, and can be set:
    - performing the [manual injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection) of the sidecar with log level properly configured inside `inject-values.yaml`.
    - via istioctl proxy-config on a specific proxy already deployed: `istioctl pc log pod_name.<namespace> --level wasm:trace`.
- **kubectl logs**: reading the logs provided by kubectl from the istio-proxy container is the main source of logs. To analyze burst of traffic it is possible to redirect the output directly to a file: `kubectl logs -f pod_name -c istio-proxy -n namespace_name > logs.txt`. 
- **dmesg from istio-proxy pod**: executed from inside the istio-proxy container, `dmesg` command may provide some hints about crashes.

- **Monitor resources via**:
    - `crictl stats` directly providing the id of the sidecar container.
    - `Grafana Dashboard` exposing the service [with Istio](https://istio.io/latest/docs/tasks/observability/metrics/using-istio-dashboard/) and analyzing the pre-made **Wasm Extension Dashboard**.


## Useful references
* WebAssembly for Proxies C++ SDK ([link](https://github.com/proxy-wasm/proxy-wasm-cpp-sdk)) and its documentation ([link](https://github.com/proxy-wasm/proxy-wasm-cpp-sdk/blob/master/docs/wasm_filter.md)).
* WASM C++ host implementation ([link](https://github.com/proxy-wasm/proxy-wasm-cpp-host)).
* Must watch talk and slides about extending Envoy with WASM ([link](https://events.istio.io/istiocon-2021/sessions/extending-envoy-with-wasm-from-start-to-finish/)).
* Libmosecurity basic examples ([link](https://github.com/esnible/wasm-examples)).
* Issue discussing libModsecurity WASM implementation ([link](https://github.com/envoyproxy/envoy/issues/17463)).


## Current Limitations
* Modsecurity has been built with minimal dependencies. Optional libraries, and relative features, are therefore not available.
* The ABI [currently does not support](https://github.com/proxy-wasm/proxy-wasm-cpp-host/issues/127) access to the file system. All the CRS rules relying on `.data` files, are currenty excluded. [This file](/basemodsec/modsec_rules_collection/excluded%20rules.txt) shows the complete list of the currently excluded rules.