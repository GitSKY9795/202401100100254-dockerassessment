# FestPass Docker & Container Security Assessment

## Personalisation

| Field | Value |
|---|---|
| Name | SUMEET KUMAR YADAV |
| REGNO | `202401100100254` |
| PORT | `134` |
| Docker Hub image | [dockskyadav/ca2-docker-assesment](https://hub.docker.com/r/dockskyadav/ca2-docker-assesment) |

The port is `80 + the last two digits of REGNO`. Create a local `.env` from
[.env.example](./.env.example); `.env` is intentionally ignored by Git.

## Run the stack

```bash
cp .env.example .env
docker compose pull
docker compose up -d
curl http://localhost:134/api/health
```

The frontend is on `frontend-backend`; only the backend is also on
`backend-db`, so the frontend cannot resolve or reach the database. The
`festpass-data` volume preserves registrations across restarts.

## Build, scan, version, and sign

```bash
docker build -t fest-backend:0.1 backend
docker run -d --name backend -p 134:5000 -e REGNO=202401100100254 fest-backend:0.1
curl http://localhost:134/api/health
docker logs backend
docker rm -f backend

trivy image fest-backend:0.1 | tee reports/trivy-before.txt
docker build --build-arg APP_VERSION=dev -t fest-backend:0.2 backend
trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 fest-backend:0.2
echo $?
trivy image fest-backend:0.2 | tee reports/trivy-after.txt

docker build --build-arg APP_VERSION=1.0.0 -t dockskyadav/ca2-docker-assesment:1.0.0 -t dockskyadav/ca2-docker-assesment:1.0 backend
docker push dockskyadav/ca2-docker-assesment:1.0.0
docker push dockskyadav/ca2-docker-assesment:1.0
docker inspect --format='{{index .RepoDigests 0}}' dockskyadav/ca2-docker-assesment:1.0.0

cosign generate-key-pair
cosign sign --key cosign.key dockskyadav/ca2-docker-assesment@DIGEST
cosign verify --key cosign.pub dockskyadav/ca2-docker-assesment@DIGEST
docker build --build-arg APP_VERSION=1.0.1 -t dockskyadav/ca2-docker-assesment:1.0.1 backend
docker push dockskyadav/ca2-docker-assesment:1.0.1
cosign verify --key cosign.pub dockskyadav/ca2-docker-assesment:1.0.1
```

Replace `DIGEST` with the immutable `sha256` digest printed by the `1.0.0`
push. Never commit `cosign.key` or `.env`.

The published `1.0.0` image is signed and verified with
[cosign.pub](./cosign.pub). The local stack is running at
`http://localhost:134`.

## Vulnerability results

| Image | CRITICAL | HIGH | Size |
|---|---:|---:|---:|
| `fest-backend:0.1` | 234 | 2,243 | 1.61 GB |
| `fest-backend:0.2` | 9 | 92 | 216 MB |

The `fest-backend:0.2` full scan contains 9 CRITICAL findings. The
`--ignore-unfixed` filtered report contains 0 CRITICAL findings in its final
summary, but the required exit-code check returned `1` because 7 fixable
CRITICAL findings were still present. These results are retained as the
assessment scan evidence.

**1.0.0 digest:** `sha256:573cb5f3a7b89aefcb127e78cc4dfcb0171cf0ffa9ab1f30d717c9fe94d337d0`

## Stage commit history

The final Git history records the assessment stages in order:

1. `Stage 1 done` - backend containerization
2. `Stage 2 done` - Compose stack and network isolation
3. `Stage 3 done` - Trivy reports and remediation work
4. `Stage 4 done` - versioned Docker Hub images
5. `Stage 5 done` - Cosign key and signature verification
