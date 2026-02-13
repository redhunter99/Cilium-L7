# Cilium-L7
use it for Layer 7 test cases



Using Hubble, check the traffic that is routed through the Envoy proxy:

shell

copy

run
hubble observe --type trace:to-proxy
Note
The trace:to-proxy filter will show all flows that go through a proxy. In general, this could be either Envoy or the Cilium DNS proxy. However, since we have not yet deployed a DNS Network Policy, you will only see flows related to Envoy at the moment.

Let's extract all the flow information based on the protocol (e.g. HTTP) and source pod (e.g. the Tie Fighter), then export the result with the JSON output option, and finally filter with jq to only see the .flow.l7 field. This will show us the specific details parsed from the L7 traffic, such as the method and headers:

shell

copy

run
hubble observe --protocol http --from-pod default/tiefighter -o jsonpb | \
  head -n 1 | jq '.flow.l7'
Observe the details of the flow, in particular the envoy-specific headers added to the request:

X-Envoy-Internal
X-Request-Id
Then, look for replies (using --to-pod instead of --from-pod this time):

shell

copy

run
hubble observe --protocol http --to-pod default/tiefighter -o jsonpb | \
  head -n 1 | jq '.flow.l7'
Observe the envoy headers:

X-Envoy-Upstream-Service-Time
X-Request-Id
All these flows are ingress flows, as you can see by filtering for HTTP flows in the egress direction, which should return nothing:

shell

copy

run
hubble observe --protocol http --traffic-direction egress
Let's use the X-Request-Id to match a request and its response.

First, we'll need to make sure egress traffic from the Tie Fighter is captured by Envoy, so we'll need a L7 CNP for that.

If we apply an egress CNP though, this will disrupt DNS requests, which are also egress traffic, so we need to add a DNS policy as well:

shell

copy

run
kubectl apply -f policies/dns.yaml -f policies/tiefighter.yaml
Now that this is applied, check for egress traffic again:

shell

copy

run
hubble observe --protocol http --traffic-direction egress
You can now see egress requests from the Tie Fighter being forwarded to the Death Star, as well as the responses from the Death Star:

Mar  5 17:54:58.107: default/tiefighter:35338 (ID:8279) -> default/deathstar-7848d6c4d5-5tndv:80 (ID:27726) http-request FORWARDED (HTTP/1.1 POST http://deathstar.default.svc.cluster.local/v1/request-landing)
Mar  5 17:54:58.109: default/tiefighter:35338 (ID:8279) <- default/deathstar-7848d6c4d5-5tndv:80 (ID:27726) http-response FORWARDED (HTTP/1.1 200 0ms (POST http://deathstar.default.svc.cluster.local/v1/request-landing))
Note
When using Kubernetes Network Policies, responses are automatically allowed and do not require an explicit rule.

This is why egress traffic corresponding to the response from the Death Star to the Tie Fighter is allowed, even though there is not egress policy for it.

Now, let's match request IDs! Run the following command to record some Hubble HTTP flows and save them to a file:

shell

copy

run
hubble observe --namespace default --protocol http -o jsonpb > flows.json
Find the first EGRESS flow in the file and get its ID:

shell

copy

run
REQUEST_ID=$(cat flows.json | jq -r '.flow | select(.source.labels[0]=="k8s:app.kubernetes.io/name=tiefighter" and .traffic_direction=="EGRESS") .l7.http.headers[] | select(.key=="X-Request-Id") .value' | head -n1)
echo $REQUEST_ID
Then find all flows with this request ID in the file and display their source identities:

shell

copy

run
cat flows.json | \
  jq 'select(.flow.l7.http.headers[] | .value == "'$REQUEST_ID'") .flow | {src_label: .source.labels[0], dst_label: .destination.labels[0], traffic_direction, type: .l7.type, time}'
You will see 4 sources:

an egress flow from the tiefighter to the deathstar, corresponding to the original request
the ingress flow for the same request, being forwarded from the proxy to the Death Star
another egress flow for the response, from the deathstar to the tiefighter
the corresponding ingress flow from the deathstar pod to the tiefighter
They might not appear in the same order depending on how flows were received from the Hubble relay servers on the different nodes.
