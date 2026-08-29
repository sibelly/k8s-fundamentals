## Running a self-hosted LLM in Kubernetes with vLLM

I used this [tutorial](https://www.cncf.io/blog/2026/07/16/running-a-self-hosted-llm-in-kubernetes-with-vllm/) as base.

#### But had to change the LLM and a lot of arguments, to make it work in my Mac M1 with just 8GB of ram. 😮‍💨

### Apply the resources

```
kubectl apply -f vllm.yaml
```

### Testing the model

- #### With port-forward
```
kubectl port-forward \
  -n vllm \
  service/vllm-server \
  8000:8000

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "HuggingFaceTB/SmolLM2-135M-Instruct",
    "max_tokens": 8,
    "temperature": 0,
    "messages": [
      {
        "role": "user",
        "content": "What is gravity?"
      }
    ]
  }'
```

- #### With another pod running curl
```
kubectl apply -f testing-pod.yaml

kubectl exec -it curl-client -- sh

curl http://vllm-server:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "HuggingFaceTB/SmolLM2-135M-Instruct",
    "max_tokens": 8,
    "temperature": 0,
    "messages": [
      {
        "role": "user",
        "content": "What is gravity?"
      }
    ]
  }'

```

## Fonts for my studyings

- https://huggingface.co/docs/transformers/index
- https://docs.vllm.ai/en/latest/design/paged_attention/

#### Papers
- 2023 - [Efficient Memory Management for Large Language Model Serving with PagedAttention](https://arxiv.org/pdf/2309.06180)
- 2017 - [Attention Is All You Need](https://arxiv.org/pdf/1706.03762)
- 2014 - [Neural Machine Translation by Jointly Learning to Align and Translate](https://arxiv.org/pdf/1409.0473)

#### Video
- [The Paper That Created Modern AI
](https://www.youtube.com/watch?v=jIo2ccqPnLQ)

#### Book
- [Attention Mechanisms and Transformers](https://d2l.ai/chapter_attention-mechanisms-and-transformers/queries-keys-values.html)

