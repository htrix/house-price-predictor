Trained models are added here.

Build and run the FastAPI Docker container:

```bash
docker build --no-cache -t fastapi .
docker run --rm -it -p 8000:8000 fastapi
```

Run FastAPI and Streamlit on the same Docker network:

```bash
docker network create ml-net
docker run -d --name model --network ml-net -p 8000:8000 fastapi
docker run --rm -it --name streamlit --network ml-net -p 8501:8501 -e API_URL=http://model:8000 streamlit-app
```
