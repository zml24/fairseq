# Aligned Transformer

|       Model        | IWSLT14 De-En | IWSLT14 En-De |
| :----------------: | :-----------: | :-----------: |
|    Transformer     |     34.61     |     28.31     |
|   Ours. No align   |       2       |       4       |
|   Ours. Aligned    |       6       |       7       |
| Ours. Aligned + lm |       9       |      10       |

## Installation

- PyTorch version >= 1.5.0
- Python version >= 3.6
- For training new models, you'll also need an NVIDIA GPU and NCCL
- To install fairseq and develop locally:

```shell
git clone https://github.com/zml24/fairseq
cd fairseq
pip install --editable ./

# on MacOS:
# CFLAGS="-stdlib=libc++" pip install --editable ./
```
- For faster training install NVIDIA's apex library:
```shell
git clone https://github.com/NVIDIA/apex
cd apex
pip install -v --disable-pip-version-check --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" \
    --global-option="--deprecated_fused_adam" --global-option="--xentropy" \
    --global-option="--fast_multihead_attn" ./
```

## IWSLT14 De-En

To get the binary dataset, follow [fairseq's example](https://github.com/pytorch/fairseq/tree/master/examples/translation)

```shell
cd examples/translation/
bash prepare-iwslt14.sh
cd ../..

TEXT=examples/translation/iwslt14.tokenized.de-en
fairseq-preprocess \
    --source-lang de --target-lang en \
    --trainpref $TEXT/train --validpref $TEXT/valid --testpref $TEXT/test \
    --destdir data-bin/iwslt14.tokenized.de-en \
    --workers 20
```

To reproduce a forward Transformer baseline (assume running on a P100 GPU)
```shell
fairseq-train \
    data-bin/iwslt14.tokenized.de-en \
    --arch transformer_iwslt_de_en --share-decoder-input-output-embed \
    --optimizer adam --adam-betas '(0.9, 0.98)' --clip-norm 5.0 \
    --lr 5e-4 --lr-scheduler inverse_sqrt --warmup-updates 4000 \
    --dropout 0.3 --weight-decay 0.0001 \
    --criterion label_smoothed_cross_entropy --label-smoothing 0.1 \
    --max-tokens 3584 --save-dir checkpoints/forward \
    --keep-last-epochs 10 --patience 10

fairseq-generate \
    data-bin/iwslt14.tokenized.de-en \
    --path checkpoints/forward/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet
```

To reproduce a backward Transformer baseline
```shell
fairseq-train \
    data-bin/iwslt14.tokenized.de-en -s en -t de \
    --arch transformer_iwslt_de_en --share-decoder-input-output-embed \
    --optimizer adam --adam-betas '(0.9, 0.98)' --clip-norm 5.0 \
    --lr 5e-4 --lr-scheduler inverse_sqrt --warmup-updates 4000 \
    --dropout 0.3 --weight-decay 0.0001 \
    --criterion label_smoothed_cross_entropy --label-smoothing 0.1 \
    --max-tokens 3584 --save-dir checkpoints/backward \
    --keep-last-epochs 10 --patience 10

fairseq-generate \
    data-bin/iwslt14.tokenized.de-en -s en -t de \
    --path checkpoints/backward/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet
```

To reproduce a bidirectional Transformer
```shell
fairseq-train --task aligned_translation \
    data-bin/iwslt14.tokenized.de-en \
    --arch aligned_transformer_iwslt_de_en --share-decoder-input-output-embed \
    --optimizer adam --adam-betas '(0.9, 0.98)' --clip-norm 5.0 \
    --lr 5e-4 --lr-scheduler inverse_sqrt --warmup-updates 4000 \
    --dropout 0.3 --weight-decay 0.0001 \
    --criterion bidirectional_label_smoothed_cross_entropy --label-smoothing 0.1 \
    --max-tokens 3584 --save-dir checkpoints/bidirection \
    --keep-last-epochs 10 --patience 10

fairseq-generate --task aligned_translation --direction s2t \
    data-bin/iwslt14.tokenized.de-en \
    --path checkpoints/bidirection/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet

fairseq-generate --task aligned_translation --direction t2s \
    data-bin/iwslt14.tokenized.de-en \
    --path checkpoints/bidirection/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet
```

To reproduce a bidirectional Transformer with alignment
```shell
fairseq-train --task aligned_translation \
    data-bin/iwslt14.tokenized.de-en \
    --arch aligned_transformer_iwslt_de_en --share-decoder-input-output-embed \
    --optimizer adam --adam-betas '(0.9, 0.98)' --clip-norm 5.0 \
    --lr 5e-4 --lr-scheduler inverse_sqrt --warmup-updates 4000 \
    --dropout 0.3 --weight-decay 0.0001 \
    --criterion bidirectional_label_smoothed_cross_entropy_with_mse --label-smoothing 0.1 \
    --max-tokens 3584 --save-dir checkpoints/alignment \
    --keep-last-epochs 10 --patience 10

fairseq-generate --task aligned_translation --direction s2t \
    data-bin/iwslt14.tokenized.de-en \
    --path checkpoints/alignment/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet

fairseq-generate --task aligned_translation --direction t2s \
    data-bin/iwslt14.tokenized.de-en \
    --path checkpoints/alignment/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet
```

To reproduce a bidirectional Transformer with alignment and better decoder
```shell
fairseq-train --task aligned_translation \
    data-bin/iwslt14.tokenized.de-en \
    --arch aligned_transformer_iwslt_de_en --share-decoder-input-output-embed \
    --optimizer adam --adam-betas '(0.9, 0.98)' --clip-norm 5.0 \
    --lr 5e-4 --lr-scheduler inverse_sqrt --warmup-updates 4000 \
    --dropout 0.3 --weight-decay 0.0001 \
    --criterion bidirectional_label_smoothed_cross_entropy_with_mse --label-smoothing 0.1 \
    --max-tokens 3584 --save-dir checkpoints/alignment_lm \
    --keep-last-epochs 10 --patience 10

# if you have trained a bidirectional Transformer with alignment
# cp -r checkpoints/alignment checkpoints/alignment_lm

fairseq-train --task aligned_translation \
    data-bin/iwslt14.tokenized.de-en \
    --arch aligned_transformer_iwslt_de_en --share-decoder-input-output-embed \
    --optimizer adam --adam-betas '(0.9, 0.98)' --clip-norm 5.0 \
    --lr 5e-4 --lr-scheduler inverse_sqrt --warmup-updates 4000 \
    --dropout 0.3 --weight-decay 0.0001 \
    --criterion bidirectional_label_smoothed_cross_entropy_lm --label-smoothing 0.1 \
    --max-tokens 3584 --save-dir checkpoints/alignment_lm \
    --keep-last-epochs 10 --patience 10

fairseq-train --task aligned_translation \
    data-bin/iwslt14.tokenized.de-en \
    --arch aligned_transformer_iwslt_de_en --share-decoder-input-output-embed \
    --optimizer adam --adam-betas '(0.9, 0.98)' --clip-norm 5.0 \
    --lr 5e-4 --lr-scheduler inverse_sqrt --warmup-updates 4000 \
    --dropout 0.3 --weight-decay 0.0001 \
    --criterion bidirectional_label_smoothed_cross_entropy --label-smoothing 0.1 \
    --max-tokens 3584 --save-dir checkpoints/alignment_lm \
    --keep-last-epochs 10 --patience 10

fairseq-generate --task aligned_translation --direction s2t \
    data-bin/iwslt14.tokenized.de-en \
    --path checkpoints/alignment_lm/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet

fairseq-generate --task aligned_translation --direction t2s \
    data-bin/iwslt14.tokenized.de-en \
    --path checkpoints/alignment_lm/checkpoint_best.pt \
    --batch-size 128 --beam 5 --remove-bpe --quiet
```
