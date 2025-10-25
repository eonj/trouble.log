Trouble ID `2025-10-23.cloudflare-api-dns-record-edit-etc`

# Cloudflare API 실패 메시지 &ldquo;PUT method not allowed for the api\_token authentication scheme&rdquo;

[Cloudflare API](<https://developers.cloudflare.com/api/>) 문서를 보자. 짐작하건대 어쩌면 가장 많은 사람이 쓰고 있지 않을까 싶은 게 있는데 바로 [DNS Records 부분이다.](<https://developers.cloudflare.com/api/resources/dns/subresources/records/>) 사용량이 가장 많지는 않겠지만. 나처럼 아직도 홈 서버를 두고 사는 사람에게는 TTL 120 정도 걸어 놓고 DDNS 운용을 하는 건 현실적인 필요를 충족하는 수준이 된다. 그 이상의 요구사항이라면 아마 [cloudflared](<https://github.com/cloudflare/cloudflared>) 같은 걸 고려하면 될 테고.

이때 보통 네트워크 구성은, 공인 IP 주소를 DHCP 로 받는 노드가 홈 서버 자체인 경우도 있으나 라우터인 경우도 많으며, 후자라면 홈 서버는 라우터의 서브넷에 있게 될 테다. 서브넷 내의 L3/L4 라우팅은 알아서 한다 쳐도 결국 홈 서버에서 서브넷에 붙은 NIC 정도 가지고는 알 수 없는 공인 IP 주소 같은 게 필요하다. 물론 이 부분은 이전부터 [ipify](<https://www.ipify.org/>) 같은 곳을 통해 문제 없이 하고 있었다. 그리고 Cloudflare API 토큰이 필요하며, DNS 레코드 ID 가 필요하다. 존 ID 는 이미 정해져 있기 때문에, API 문서에 `GET /zones/{zone_id}/dns_records` 라고 되어 있는 것을 호출하면, 내가 원하는 서브도메인 A 항목의 레코드 ID 를 알 수 있다. 공통 base URL 을 적용해, 셸 스크립트 수준에서도 쉽게 이를 얻는다.

```sh
curl -X GET "https://api.cloudflare.com/client/v4/zones/$zone_id/dns_records" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $api_token" \
  > response.get_dns_records.json
```

그러면 필요한 정보가 모두 있으니 업데이트만 하면 된다. 둘 중 무엇을 쓰든 괜찮다. &ldquo;Update DNS Record&rdquo; (`PATCH /zones/{zone_id}/dns_records/{dns_record_id}`) 그리고 &ldquo;Overwrite DNS Record&rdquo; (`PUT /zones/{zone_id}/dns_records/{dns_record_id}`) 가 있다. 후자를 써 보자. `ARecord` 유형 명세에 맞는 `$request_a_record_json` 준비하시고.

```sh
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$zone_id/dns_records/$dns_record_id" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $api_token" \
  -d "$request_a_record_json"
  > response.put_dns_record.json
```

아무튼 이 방법을 쓰면, 응답 본문을 JSON 으로 받을 수 있다. 이렇게.

```json
{"success":false,"errors":[{"code":10000,"message":"PUT method not allowed for the api_token authentication scheme"}]}
```

아니 뭐 API 토큰을 쓰는 게 아니면 인증 키를 (`X-Auth-Key`, `X-Auth-Email`) 쓰란 말인가? 레거시인데? 아니면 `PUT` 이 잘못되었으니 `PATCH` 를 써야 하나 &mdash; 문서에는 둘 다 있는데? 어림없지. PATCH method not allowed for the api\_token authentication scheme.

이 문제의 원인은 하나뿐이다. 코드 상에 레코드 ID 가 없어서 요청의 경로가 `/client/v4/zones/(zone_id)/dns_records/` 로 끝나고 만 것이다. 이는 처음에 접근한 경로인 `GET /zones/(zone_id)/dns_records` 와 같다. 이 경로에 대한 연산은 &ldquo;List DNS Records&rdquo; (`GET ...`) 또는 &ldquo;Create DNS Record&rdquo; (`POST ...`) 뿐이다. `PUT` 은 허용되지 않는다. 끝.

진단? 이런 문제가 생기는 내재적 원인은, 물론 다양하겠지만, 예컨대 `response.get_dns_records.json` 을 파싱하지 못하는 것일 수 있다. 셸 스크립트의 경우 `jq` 같은 걸 사용했을 법하고 이게 PATH 범위의 명령어에 없었거나 아무튼 예상되지 않은 동작을 했을 수 있다. 물론 이 문제는 ISP DHCP 에서 주는 공인 IP 주소가 변경되고 나서 발견된다. 어느 정도 새너티 체크가 있더라도 모니터링이 없다면 미리 대응할 수 없으며, 트러블슈팅은 사람 몫이다. 규모의 경제에서 그 비용이 희석될 뿐.
