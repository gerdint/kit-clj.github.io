Kit uses [Reitit](https://metosin.github.io/reitit/) to define application routes. The routes are the entry points to your application and are used to establish a communication protocol between the server and the client.

### Routes

Reitit route handlers are just functions that
[accept request maps and return response maps](https://github.com/mmcgrana/ring/blob/master/SPEC).
Routes are defined as vectors of String path and optional (non-sequential) route argument child routes.
A route is a mapping from a URL pattern to a map containing the handlers keyed on the request method:

```clojure
["/" {:get (fn [request] {:status 200 :body "GET request"})
      :post (fn [request] {:status 200 :body "POST request"})}]
```

The body may be a function, that must accept the request as a parameter:

```clojure
(fn [request] {:status 200 :body (keys request)})
```

The above route reads out all the keys from the request map and displays them.
The output will look like the following:

```clojure
["reitit.core/match","reitit.core/router","ssl-client-cert","cookies","remote-addr","params","flash","handler-type","headers","server-port","muuntaja/request","content-length","form-params","server-exchange","query-params","content-type","path-info","character-encoding","context","uri","server-name","anti-forgery-token","query-string","path-params","muuntaja/response","body","multipart-params","scheme","request-method","session"]
```

Reitit supports three kinds of parameters. These can be route, query, and body parameters.
For example, if we create the following route:

```clojure
["/foo/:bar" {:post (fn [{:keys [path-params query-params body-params]}]
                        {:status 200
                         :body   (str "path params: " path-params
                                      "\nquery params: " query-params
                                      "\nbody params: " body-params)})}]
```

Then we could query it via cURL:

```
curl --header "Content-Type: application/json" \
--request POST \
--data '{"username":"xyz","password":"xyz"}' \
'localhost:3000/foo/bar?foo=bar'
```

and the params will be parsed out as follows:

```clojure
path params: {:bar "bar"}
query params: {"foo" "bar"}
body params: {:password "xyz", :username "xyz"}
```

In the guestbook application example we saw the following route defined:

```clojure
["/" {:get home-page
      :post save-message!}]
```

This route serves the home page when it receives a `GET` request and extracts the name and the message form parameters when it receives a `POST` request.
Note that `POST` requests must contain a CSRF token by default. This is handled by the `middleware/wrap-csrf` declaration below:

```clojure
(defn home-routes [base-path]
  [base-path
   ["/" {:get home-page
         :post save-message!}]])
```

Please refer [here](/docs/security.html#cross_site_request_forgery_protection) for more details on managing CSRF middleware.

### Return values

The return value of a route block determines at least the response body
passed on to the HTTP client, or at least the next middleware in the
ring stack. Most commonly, this is a string, as in the above examples.
But, we may also return a [response map](https://github.com/ring-clojure/ring/blob/master/SPEC):

```clojure
["/" {:get (fn [request] {:status 200 :body "Hello World"})}]

["/is-403" {:get (fn [request] {:status 403 :body ""})}]

["/is-json" {:get (fn [request] {:status 200 :headers {"Content-Type" "application/json"} :body "{}"})}]
```

## Static Resources

By default, any resources located under the `resources/public` directory will be available to the clients.
This is handled by the `reitit.ring/resource-handler` handler found in the `<app>.handler` namespace:

```clojure
(ring/create-resource-handler {:path "/"})
```

Any resources found on the classpath of the application can be accessed using `clojure.java.io/resource` function:

```clojure
(slurp (clojure.java.io/resource "myfile.md"))
```

Conventionally, non-source resources should be placed in the `resources` directory of the project.

### Handling file uploads

Given a page called `upload.html` with the following form:

```xml
<h2>Upload a file</h2>
<form action="/upload" enctype="multipart/form-data" method="POST">
    {% csrf-field %}
    <input id="file" name="file" type="file" />
    <input type="submit" value="upload" />
</form>
```

we could then render the page and handle the file upload as follows:

```clojure
(ns myapp.upload
  (:require [myapp.layout :as layout]
            [ring.util.response :refer [redirect file-response]])
  (:import [java.io File FileInputStream FileOutputStream]))

(def resource-path "/tmp/")

(defn file-path [path & [filename]]
  (java.net.URLDecoder/decode
    (str path File/separator filename)
    "utf-8"))

(defn upload-file
  "uploads a file to the target folder
   when :create-path? flag is set to true then the target path will be created"
  [path {:keys [tempfile size filename]}]
  (try
    (with-open [in (new FileInputStream tempfile)
                out (new FileOutputStream (file-path path filename))]
      (let [source (.getChannel in)
            dest   (.getChannel out)]
        (.transferFrom dest source 0 (.size source))
        (.flush out)))))

(defn home-routes [base-path]
  [base-path
   ["/upload" {:get (fn [req]
                      (layout/render request "upload.html"))

               :post (fn [{{:keys [file]} :params}]
                       (upload-file resource-path file)
                       (redirect (str "/files/" (:filename file))))}]

   ["/files/:filename" {:get (fn [{{:keys [filename]} :path-params}]
                               (file-response (str resource-path filename)))}]])
```

Th `:file` request form parameter points to a map containing the description of the file that will be uploaded. Our `upload-file` function above uses `:tempfile`, `:size` and `:filename` keys from this map to save the file on disk.

A file upload progress listener can be added in the `<app>.middleware/wrap-base` function by updating `wrap-defaults` as follows:

```clojure
(wrap-defaults
  (-> site-defaults
      (assoc-in [:security :anti-forgery] false)
      (dissoc :session)
      (assoc-in [:params :multipart]
                {:progress-fn
                 (fn [request bytes-read content-length item-count]
                   (log/info "bytes read:" bytes-read
                             "\ncontent length:" content-length
                             "\nitem count:" item-count))})))
```

Alternatively, if you're fronting with Nginx then you can use its [Upload Progress Module](http://wiki.nginx.org/HttpUploadProgressModule).

## Organizing application routes

It's a good practice to organize your application routes together by functionality.
Your application will typically have two types of routes. The first type is used to serve
HTML pages that are rendered by the browser. The second type are routes used to expose your
service API. These are accessed by the client to retrieve data from the server via AJAX.

The route groups are defined using Integrant components as follows:

```clojure
(derive :reitit.routes/api :reitit/routes)

(defmethod ig/init-key :reitit.routes/api
  [_ {:keys [base-path]
      :or   {base-path ""}
      :as   opts}]
  [base-path (route-data opts) (api-routes opts)])
```

All the routing components that derive `:reitit/routes` are aggregated by the `:router/routes` component:

```clojure
(defmethod ig/init-key :router/routes
  [_ {:keys [routes]}]
  (apply conj [] routes))
```

Finally, you'll notice that Ring handler uses the router defined using
the `:router/routes` Integrant multimethod above. This component is referenced by the router component in `reosurces/system.edn`
under the `router` key:

```clojure
:handler/ring
 {:router #ig/ref :router/core
  ...}
```

The Integrant intitialized multimethod for `:handler/ring` then uses the `:router` key from the provided options:

```clojure
(defmethod ig/init-key :handler/ring
  [_ {:keys [router api-path] :as opts}]
  (ring/ring-handler
    router
    (ring/routes
      (when (some? api-path)
        (swagger-ui/create-swagger-ui-handler {:path api-path
                                               :url  (str api-path "/swagger.json")}))
      (ring/create-default-handler
        {:not-found
         (constantly {:status 404, :body "Page not found"})
         :method-not-allowed
         (constantly {:status 405, :body "Not allowed"})
         :not-acceptable
         (constantly {:status 406, :body "Not acceptable"})}))
    {:middleware [(middleware/wrap-base opts)]}))
```

Further documentation is available on the [official Reitit documentation](https://metosin.github.io/reitit/)

## Restricting access

Some pages should only be accessible if specific conditions are met. For example,
you may wish to define admin pages only visible to the administrator, or a user profile
page which is only visible if there is a user in the session.

### Restricting access based on route groups

The simplest way to restrict access is by applying the `restrict` middleware from [buddy-auth](https://github.com/funcool/buddy-auth) to
groups of routes that should not be publicly accessible.
First, we'll add the following code in the `<app>.middleware` namespace:

```clojure
(ns <app>.middleware
  (:require
    ...
    [buddy.auth.middleware :refer [wrap-authentication]]
    [buddy.auth.backends.session :refer [session-backend]]
    [buddy.auth.accessrules :refer [restrict]]
    [buddy.auth :refer [authenticated?]]))

(defn on-error [request response]
  {:status  403
   :headers {"Content-Type" "text/plain"}
   :body    (str "Access to " (:uri request) " is not authorized")})


(defn wrap-restricted [handler]
  (restrict handler {:handler authenticated?
                     :on-error on-error}))

(defn wrap-base
  [{:keys [metrics site-defaults-config cookie-secret] :as opts}]
  (let [cookie-store (cookie/cookie-store {:key (.getBytes ^String cookie-secret)})]
    (fn [handler]
      (cond-> ((:middleware env/defaults) handler opts)
        true (wrap-authentication (session-backend))
        true (defaults/wrap-defaults
              (assoc-in site-defaults-config [:session :store] cookie-store))))))            

```

We'll wrap the authentication middleware that will set the `:identity` key in the request if it's present in the session.
The session backend is the simplest one available, however Buddy provides a number of different authentications backends
as described [here](https://funcool.github.io/buddy-auth/latest/#_authentication).

The `authenticated?` helper is used to check the `:identity` key in the request and pass it to the handler when it's present.
Otherwise, the `on-error` function will be called.

We can now wrap the route groups we wish to be private using the `wrap-restricted` middleware in the `api` namespace as follows:

```clojure
(defn route-data
  [opts]
  (merge
    opts
    {:coercion   malli/coercion
     :muuntaja   formats/instance
     :swagger    {:id ::api}
     :middleware [...
                  ;; wrap restricted routes
                  wrap-restricted]}))
```

For more information on using middleware with Reitit, please see the official documentation [here](https://cljdoc.org/d/metosin/reitit/0.7.0-alpha7/doc/ring/data-driven-middleware).


### Restricting access based on URI

Using the `buddy.auth.accessrules` namespace from [Buddy](https://funcool.github.io/buddy-auth/latest/), we can define rules for restricting access to specific pages based on its URI pattern.

### Specifying Access Rules

Let's take a look at how to create a rule to specify that restricted routes should only be
accessible if the `:identity` key is present in the session.

First, we'll reference several Buddy namespaces in the `<app>.middleware` namespace.

```clojure
(ns myapp.middleware
  (:require ...
            [buddy.auth.middleware :refer [wrap-authentication]]
            [buddy.auth.accessrules :refer [wrap-access-rules]]
            [buddy.auth.backends.session :refer [session-backend]]
            [buddy.auth :refer [authenticated?]]))
```

Next, we'll create the access rules for our routes. The rules are defined using a vector where each rule is represented using a map. A simple rule that checks whether the user has been authenticated can be seen below.

```clojure
(def rules
  [{:uri "/restricted"
    :handler authenticated?}])
```

We'll also define an error handler function that will be used when access to a particular route is denied:

```clojure
(defn on-error
  [request value]
  {:status 403
   :headers {}
   :body "Not authorized"})
```

Finally, we have to add the necessary middleware to enable the access rules and authentication using a session backend.

```clojure
(defn wrap-base
  [{:keys [metrics site-defaults-config cookie-session] :as opts}]
  (let [cookie-store (cookie/cookie-store {:key (.getBytes ^String cookie-secret)})]
    (fn [handler]
      (cond-> ((:middleware env/defaults) handler opts)
        true (wrap-access-rules {:rules rules :on-error on-error})
        true (wrap-authentication (session-backend))
        true (defaults/wrap-defaults
              (assoc-in site-defaults-config [:session :store] cookie-store))))))
```

Note that the order of the middleware matters and `wrap-access-rules` must precede `wrap-authentication`.

Buddy session based authentication is triggered by setting the `:identity` key in the session when the user is successfully authenticated.

```clojure
(def user {:id "bob" :pass "secret"})

(defn login! [{:keys [params session]}]
  (when (= user params)
    (-> "ok"
        response
        (content-type "text/html")
        (assoc :session (assoc session :identity "foo")))))
```
