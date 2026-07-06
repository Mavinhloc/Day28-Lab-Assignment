one gateway, one URL, path-based routing (pure ngrok, recommended)
Run a small proxy on port 8000 and tunnel only that:
python# gateway.py — run this in a thread like your other servers
from fastapi import FastAPI, Request
from fastapi.responses import Response
import httpx, uvicorn

app = FastAPI()
client = httpx.AsyncClient(timeout=None)

async def proxy(port, path, request):
    r = await client.request(
        request.method, f"http://localhost:{port}/{path}",
        content=await request.body(),
        headers={k: v for k, v in request.headers.items() if k.lower() != "host"},
    )
    return Response(content=r.content, status_code=r.status_code,
                    media_type=r.headers.get("content-type"))

@app.api_route("/vllm/{path:path}", methods=["GET", "POST"])
async def vllm(path: str, request: Request):
    return await proxy(8001, path, request)

@app.api_route("/embed/{path:path}", methods=["GET", "POST"])
async def embed(path: str, request: Request):
    return await proxy(8002, path, request)

def run_gateway():
    uvicorn.run(app, host="0.0.0.0", port=8000)
pythonthreading.Thread(target=run_gateway, daemon=True).start()
tunnel = ngrok.connect(8000, "http")
print(tunnel.public_url)
Your .env becomes VLLM_URL={url}/vllm/v1 and EMBED_URL={url}/embed. Bonus: both URLs are permanently stable since the dev domain never changes — no more copying into .env every run.