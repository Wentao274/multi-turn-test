# multi-turn-test
multi turn test for maas


python benchmark_serving_multi_turn.py \
--model /data/models/MiniMax-M2.5-W8A8 \
--served-model-name "minimax-m2.5" \
--url "http://127.0.0.1:8080/v1" \
--input-file "generate_conversations.json" \
--output-file "generate_conversations_output.json" \
--num-clients 50 \
--max-active-conversations 100 \
--warmup-step \
--extra-body-json '{"chat_template_kwargs":{"enable_thinking":false}}' 