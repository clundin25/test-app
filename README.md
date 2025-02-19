# 1. overview

`client` folder contains a python HTTP sample and a Java grpc sample to talk to pubsub mtls endpoint via auth proxy.

`proxy` folder contains:
- `certs` folder: the CA cert used by the `auth_proxy.go`. The cert file will be regenerated by the auth proxy when we start it.
- `auth_proxy.go`: the auth proxy. Which sits in client's local machine, and connects between client and server. The auth proxy is configurable using the `auth_proxy_config.json` file.
- `auth_proxy_config.json`: the configuration file for auth proxy. It can be configured to attach auth tokens, set up mTLS transport with enterprise cert, and talk to server via customer's proxy.
- `customer_proxy.go`: a simple pass through proxy for testing the customer proxy use case. It runs at http://127.0.0.1:8888

# 2. Generate Certificates for Auth Proxy

**NOTE**: This command requires OpenSSL.

```bash
$ ./proxy/certs/generate_cert.sh
```

# 3. Run the proxy

First start the customer proxy in `proxy` folder in a new terminal by running
```
go run -v customer_proxy.go
```

Then start the auth proxy in `proxy` folder in a new terminal by running
```
go run -v auth_proxy.go
```

# 4. Test with the python app

Open a new terminal in `client/python` folder.

Run the following with "sijunliu@beyondcorp.us" account and `sijunliu-dca-test` project.

```
gcloud auth application-default login
```

Next install the dependencies.
```
python -m pip install -r requirements.txt
```

Then run
```
python app_no_cred.py
```

You can make sure the customer proxy is used by looking at the log.

# 5. Test with gcloud

First run the following with "sijunliu@beyondcorp.us" account and `sijunliu-dca-test` project.

```
gcloud auth login
gcloud config set project sijunliu-dca-test
```

Then set up gcloud as follows to use auth proxy

```
gcloud config set proxy/type http
gcloud config set proxy/address localhost
gcloud config set proxy/port 9999
gcloud config set core/custom_ca_certs_file <ca cert path>
```
The CA cert path is the absolute path of `proxy/certs/ca_cert.pem`, in my case, `/usr/local/google/home/sijunliu/wks/proxy/test-app/proxy/certs/ca_cert.pem`.

Next enable mtls for gcloud. Note that gcloud will auto switch to mtls endpoint, and this is all we need (we don't actually need the mtls connection since the auth proxy doesn't check the client cert).

```
gcloud config set context_aware/use_client_certificate true
```

Next run
```
gcloud pubsub topics list
```

# 6. Test with Java sample

Navigate to `client/java-sample` folder, run

```
export GRPC_PROXY_EXP=127.0.0.1:9999
export GOOGLE_API_USE_MTLS_ENDPOINT="always"
mvn compile
mvn exec:java -Dexec.mainClass="com.mycompany.app.App"
```
