# Response Headers

This guide describes the headers that {{ PRODUCT_NAME }} injects into responses, making them visible to your client code.

## General Headers

- `{{ HEADER_PREFIX }}-version`: version fingerprint that includes {{ PRODUCT_NAME }} version number, site build number and UTC timestamp of the build
- `{{ HEADER_PREFIX }}-t`: telemetry measurements for all the components in {{ PRODUCT_NAME }} critical path that served your request
- `{{ HEADER_PREFIX }}-request-id`: the unique ID of the request on {{ PRODUCT_NAME }} infrastructure
- `{{ HEADER_PREFIX }}-hit-request-id`: the unique ID of the request whose cached response is being returned (not present if cache miss)
- `{{ HEADER_PREFIX }}-caching-status`: indicates why a response was or was not cached. See [Caching](/guides/caching#section_why_is_my_response_not_being_cached_).
- `{{ HEADER_PREFIX }}-surrogate-key`: a space separated list of secondary cache keys used for [cache clearing](/guides/purging#surrogate_keys)

### Structure of `{{ HEADER_PREFIX }}-t`

The format is `{{ HEADER_PREFIX }}-t: <id>=<time>[,<id2>=<time2>...]`

`{{ HEADER_PREFIX }}-t` is an ordered list of telemetry measurements; values are **prepended** at response time. Thus, from left to right, measurements are ordered from the outermost edge component to the innermost cloud component that handled the request.

All times are in milliseconds.

The following structure is important to note when reading the telemtry data:

All POPs have the same components: 
* HAProxy -> Varnish -> DPS
* L1 is Edge w/ HAProxy -> Varnish -> DPS -> Global POP
* L2 is Global w/ HAProxy -> Varnish -> DPS  -> backend (user defined backend from [layer0.config](https://docs.layer0.co/guides/layer0_config#section_backends) | [static page](https://docs.layer0.co/guides/static_sites#section_router_configuration) | XBP->[Serverless](https://docs.layer0.co/guides/serverless_functions#section_serverless_functions))

***
**Note**: When a request is reentrant, telemetry information is not duplicated; instead, each request logs its own telemetry but does not return it to the downstream {{ PRODUCT_NAME }} request. As a result, duplicate entries are not possible. 
***


#### Component Names and Prefixes

Component names within the header are abbreviated: 

| Abbreviation | Component Name |
| ------------ | -------------- |
| eh  | HAProxy on edge POP              |
| ec  | Varnish cache on edge POP        |
| ed  | DPS on edge POP                  |
| gh | HAProxy on global POP            |
| gc | Varnish cache on global POP      |
| gd | DPS on global POP                |
| p  | XDN Buffer Proxy                 |
| w  | Lambda workers                   |


#### Telemetry Types
| Type | Description |
| ------------ | -------------- |
| t | Total time (example: `eht`) total time as measured by edge HAProxy) |
| f | Fetch time (example: `gdf`) total fetch time time as measured by global DPS) |
| c | Cache status (example: `ecc=miss,...,gcc=hit`) miss on the edge pop, hit on the global pop |
 
### Examples
The examples below use a response that traversed from the edge, to global and to serverless:
##### _{{ HEADER_PREFIX }}-t_
`< {{ HEADER_PREFIX }}-t: eh=1160,ect=1158,ecc=miss,edt=1152,edd=0,edf=1152,gh=869,gct=866,gcc=miss,gdt=853,gdd=0,gdf=853,pt=811,pc=1,pf=809,wm=317,wt=722,wc=19,wg=746940,wl=30896,wr=1,wp=705,wz=1`

Below is a translation of each value in this example:

| Value | Description |
| -------------- | -------------- |
| `eh=1160`  | Edge POP total time of 1160ms |
| `ect=1158` | Edge POP Varnish total time of 1158ms |
| `ecc=miss` | Edge POP Cache Miss, on miss the request will go to global POP |
| `edt=1152` | Edge POP DPS total time of 1015ms |
| `edd=0`    | Edge DPS DNS lookup time of 0ms (We do DNS caching in DPS to accerate requests, so 0 is common and expected) |
| `edf=1152` | Edge DPS Fetch time of 1152ms |
| `gh=869`   | Global POP HAProxy total time of 869ms |
| `gct=866`  | Global POP Varnish total time of 866ms |
| `gcc=miss` | Global POP Cache Miss - on miss request will go to backend |
| `gdt=853`  | Global POP DPS total time of 853ms |
| `gdd=0`    | Global POP DNS Lookup time of 0ms (implying cached DNS lookup) |
| `gdf=853`  | Global POP DPS fetch time to backend of 853ms |
| `pt=811`   | XBP Total time of 811ms |
| `pc=1`     | XBP total request count. if > 1 it implies scaling where we had to queue and retry this request |
| `pf=809`   | XBP Total Fetch time to serverless of 809ms |
| `wm=317`   | Serverless worker memory used 317mb |
| `wt=722`   | Serverless total time of 722ms |
| `wc=19`    | Number of times this serverless instance has been invoked (19) |
| `wg=746940`| Age of this serverless instance of 746,940ms |
| `wl=30896` | Sum of worker times across all requests |
| `wr=1`     | Time spent evaluating route |
| `wp=705`   | Worker processing time |
| `wz=1`     | If the app contains an image optimization tag, like Next [<Image>](https://nextjs.org/docs/api-reference/next/image) or Nuxt [<nuxt-img>](https://image.nuxtjs.org/components/nuxt-img/) this will be 1 |

##### _{{ HEADER_PREFIX }}-status_
The `{{ HEADER_PREFIX }}-status` header will show the response codes received from the preceding service at each step in the process

`< {{ HEADER_PREFIX }}-status: eh=200,ed=200,gh=200,gd=200,p=200,w=200`

##### _{{ HEADER_PREFIX }}-components_
 `{{ HEADER_PREFIX }}components`. This is most useful for {{ PRODUCT_NAME }} in identifying the versions of each service, the id of the environment, and which backend serviced the request

`< {{ HEADER_PREFIX }}-components: eh=0.1.6,e=atl,ec=1.1.0,ed=1.0.1,gh=0.1.6,g=hef,gd=1.0.1,p=1.21.10,w=3.11.0,wi=e8ce8753-163d-4be9-a39e-40454ace5146,b=serverless`


## server-timing

{{ PRODUCT_NAME }} adds the following values to the standard [server-timing](https://www.w3.org/TR/server-timing/) response header:

- layer0-cache: desc=`value` - value will be one of:
  - `HIT-L1` - The page was served from the edge cache
  - `HIT-L2` - The page was served from the shield cache
  - `MISS` - The page could not be served from the cache
- country: desc=`country_code` - where country_code is the two letter code of the country from which the request was sent.
- xrj: desc=`route` - where route is the matched route serialized as JSON.

## Troubleshooting Headers

The following headers are used internally by {{ PRODUCT_NAME }} staff to troubleshoot issues with requests.

- `{{ HEADER_PREFIX }}-status`: HTTP status that each component in {{ PRODUCT_NAME }} returned (can vary depending on the component)
- `{{ HEADER_PREFIX }}-components`: version of various components in {{ PRODUCT_NAME }} critical path that serviced your request
