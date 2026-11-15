---
icon: magnifying-glass-plus
layout:
  width: default
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
  anchors:
    visible: true
---

# Transformer

A **transformer** is the design of almost every current language model: a stack of identical layers, each with an attention part and a feed-forward part (the T of GPT, from the 2017 paper _Attention Is All You Need_). This page zooms on step 3 of [how the tables connect](./#how-the-tables-connect), the inside of a layer, with Llama 3.1 8B reading "The capital of France is". Before 2017, models read a text one word after another; [History › Transformers](../../../history/overview/transformers-2017-2021.md) shows what changed.

{% hint style="info" %}
**A transformer is the shape of the model, not a training method.** The same 32 layers run when the lab trains the model and every time you use it. Inside each layer, attention is the only step where tokens exchange information, and a token reads itself and every token before it, never one that comes after.
{% endhint %}

#### Before you start

The notions this page relies on, from earlier pages, in one picture. Step 3, the 32 layers, is what a transformer is.

<figure><img src="../../../.gitbook/assets/0.transformer_ai-transformer-before_v1.png" alt=""><figcaption></figcaption></figure>

* **Steps 1 to 5:** Each token's number selects its row of the token table, a list of 4096 numbers; the 32 layers rework the lists; the output table scores every token of the vocabulary; the sampler picks one, and only that token goes round to the next turn (see [Inside a model](./) and [Text generation](../../inference/text-generation/)).
* **A, training:** The lab feeds whole texts; at every token, the percentage given to the real next token is checked, and the gaps correct the tables, billions of times; then the tables are final (see [How a model learns](../how-a-model-learns.md)).
* **B, use:** The tables are fixed. The prompt is read in one pass (prefill), then the answer comes one token per turn, and the [KV cache](../../inference/text-generation/kv-cache.md) keeps each token's keys and values so that earlier tokens are not computed again.
* **Stored or computed:** Tables are stored in the file: random before training, corrected by it, then fixed. Lists are computed for each text and dropped; the query, key and value are lists made by the tables q, k and v, and at use the keys and values are kept in the KV cache.

#### What the layers are for

To pick the next token, the output table reads only the **final list**: the list of the last column after layer 32 and the final norm. The 32 layers are there to bring the whole prompt into that list.

<figure><img src="../../../.gitbook/assets/1.transformer_ai-transformer-why_v2.png" alt=""><figcaption></figcaption></figure>



* **A, without the layers:** `list_374` is a copy of row 374 of the token table: it describes " is" in any text. Sent straight to the output table, it would give the same guess after every " is", whatever came before.
* **B, with the layers:** In each layer, attention brings information from the earlier tokens into the list (here " capital" and " France"), and the feed-forward part adds what training stored. After layer 32, `list_374` is the final list: still the list of " is", reworked by the whole prompt, so that it describes what should come next. The final norm brings its 4096 numbers to a standard scale without changing how many there are; the output table (also called the LM head; not table o, which sits inside each layer) turns them into 128,256 scores, and " Paris" gets the highest percentage.
* **C, another prompt:** In "The sky is", " is" starts from the same row 374 and goes through the same tables, but reads other tokens, so its final list differs and so do the percentages. The list is computed for each text and dropped; the weights are in the tables, which do not change at use.
* **D, why 32 layers and not one:** Each layer starts from the lists as the layers before it left them, so steps that need each other can follow one another. To give " Paris", the list of " France" first takes in what training stored about France (feed-forward, early layers), " is" picks up that a capital is asked (attention, middle layers), then reads the enriched list of " France" and takes "Paris" from it (attention, upper layers). In one layer, " is" would read " France" as it entered that layer, before its feed-forward part added anything. This order, simplified, is the one [Geva et al. (2023)](https://arxiv.org/abs/2304.14767) traced in GPT-2 and GPT-J; how many layers is a design choice, 32 in Llama 3.1 8B and 80 in 70B.
* **Who sets the tables:** They start as random numbers. Training corrects them on billions of texts until B works (see [Training, prompt and answer](transformer.md#training-prompt-and-answer)); at use they are fixed, and the same calculation gives the percentages.

In a brain, the tables would be the strength of the synapses, set by learning, and the lists the activity of the neurons while one text is read. Unlike a brain, a model learns nothing while it answers: it keeps only the text of the conversation and its KV cache.

#### The map

One layer, drawn for the five tokens of the prompt: each token climbs its own column, from 1 at the bottom to 9 at the top. Each coloured outline is a part that a later section opens.

<figure><img src="../../../.gitbook/assets/2.transformer_ai-transformer-map_v1.svg" alt=""><figcaption></figcaption></figure>

1. **Tokens:** One column per token, in order: "The" is position 1, " is" position 5.
2. **Lists:** Each column starts as a copy of its token's row of the token table: `list_791` for "The", `list_374` for " is", 4096 numbers each.
3. **attn\_norm:** Brings each list to the same scale (see [Heads](transformer.md#heads)).
4. **q, k, v:** Three tables turn each list into a **query** (what the token looks for), a **key** (what it offers) and a **value** (what it passes on). Each is a list of numbers computed from the token's list: not the token, its token number or its position.
5. **Attention:** Each column compares its query with the keys of every column up to it, from “The” to itself, and takes a mix of their values (see [One attention step](transformer.md#one-attention-step)). It is the only step where columns read each other, and none reads a later one: " France" (4) never reads " is" (5), while " is" reads " France". This rule is the **causal mask** (grid on the right).
6. **o, +:** Table o, the last table of the attention part, turns the mix into a correction, added to the list that came in (dashed grey arrow): a layer adds, it does not replace. Table o is not the output table, which comes once, after the last layer.
7. **Feed-forward, +:** ffn\_norm, then three tables (gate, up, down) work on each column alone, and the result is added again (see [Feed-forward](transformer.md#feed-forward)). Some models split this part into experts (see [Mixture of experts](mixture-of-experts.md)).
8. **32 layers:** The 31 other layers repeat 3 to 7, each with its own tables.
9. **Scores:** After the final norm, the output table (LM head) scores the last column, and " Paris" gets the highest percentage. The other columns are scored only in training, against the token that follows them in the text.

#### Training, prompt and answer

The same layers run at three moments: when the lab trains the model, when the model reads your prompt, and when it writes the answer. What changes is what goes in, how many columns run in one pass, and whether there is a right answer to check the percentages against.

<figure><img src="../../../.gitbook/assets/3.transformer_ai-transformer-when_v1.png" alt=""><figcaption></figcaption></figure>

* **A, training:** The text is known, so all its columns run in one pass, and each column's percentages are checked against the token that really follows: "The cat sat on the mat" gives 5 checks at once. The correction is not part of the pass: after it, **backpropagation** goes back from the gaps and corrects every table, the output table and the token table included, all in the same step; the lower the percentage given to the right token, the larger the correction (see [training](../how-a-model-learns.md#training)). The lists are not kept: the next pass recomputes them with the corrected tables. Before training the tables are random, so every token gets about 1 chance in 128,256; across texts "The" is followed by many words, so its column ends up spreading its percentages.
* **B, prefill:** Your prompt has no answer to check against: its columns run in one pass, and only the last one's percentages are used, by the sampler. Their keys and values fill the KV cache; if the engine still holds the cache for the same beginning of the conversation, only the new tokens are computed (see [one conversation, message by message](../../inference/text-generation/kv-cache.md#one-conversation-message-by-message)). A long prompt is read in chunks (2,048 tokens by default in llama.cpp), with the same result.
* **C, the loop:** The next token does not exist until the sampler picks it, so each turn adds one column: " Paris" reads the cached keys and values of the five columns before it and its own, then adds its own to the cache.

Training works like a fill-in-the-blank exercise with the answer key; use is the same guess without the key, so nothing is corrected. Within a pass the columns are computed in parallel (see [GPU & memory](../../inference/gpu-and-memory.md)); at position 100,000, a token's query meets 100,000 keys, in each head of each layer.

#### Heads

The attention part of a layer has 4 tables, q, k, v and o, after its norm. Between them run 32 **heads**, each comparing its own query with the keys, so that one layer can follow several relations at once. A head is not a table but a slice: 128 of the 4096 numbers of the query, with the key and the value of its group. Each of the 32 layers has its own 32 heads.

<figure><img src="../../../.gitbook/assets/4.transformer_ai-transformer-heads_v2.png" alt=""><figcaption></figcaption></figure>

* **A, attn\_norm:** The list is divided by the typical size of its numbers (the square root of the average of their squares), then each number is multiplied by its own weight in the norm table, set by training like every table. The list keeps its 4096 numbers; only their scale changes, so every list reaches q, k and v at the same scale, whatever the earlier layers added. ffn\_norm does the same before the feed-forward part, and the final norm before the output table.
* **B, q, k, v:** Each table makes its list the same way: row 1 of table q times the whole list, number by number, added, gives number 1 of q1, row 2 number 2 of q1, row 129 number 1 of q2, up to row 4096 (drawn on the left of the diagram). The tables stay fixed; each token brings its own list. The token table is the only one that is read instead, one row picked by the token number. q gives 4096 numbers, cut into 32 queries of 128: q1 is numbers 1 to 128, q2 numbers 129 to 256, and so on. k and v give 1024 each, 8 keys and 8 values of 128. The 32 queries are the 32 heads of this one layer, not one per layer: all of B happens inside blk.0, and the 31 other layers repeat it with their own tables. The queries and keys are turned by their position before the heads compare them (see [Position](transformer.md#position)); the values are not. This is why `attn_k` and `attn_v` are \[1024 × 4096] while `attn_q` is \[4096 × 4096] (see [what is inside a version](./#what-is-inside-a-version)). The 128 numbers describe something only together, like the three numbers of a colour code, except that no single one has a name: training settles what they mean.
* **Groups:** Heads 1 to 4 form group 1: head 1 is q1 with key k1 and value v1, head 2 is q2 with the same k1 and v1, and so on; heads 5 to 8 use k2 and v2, up to head 32, q32 with k8 and v8. This is **grouped-query attention** (GQA). The KV cache then keeps 8 keys and 8 values per token and layer instead of 32, 4 times less memory (see [how big it gets](../../inference/text-generation/kv-cache.md#how-big-it-gets)).
* **o:** Once its calculation is done, each head gives an output of 128 numbers, drawn under its column (out 1 to out 32; the output of head 1 is part C of One attention step). The 32 outputs are put one after the other, 4096 numbers, and table o mixes them into one list, added to the list that came in.
* **C, two heads:** Head 1 and head 5 of " is" run side by side in the same layer. Another query and other keys give other shares: head 1 takes 53 % of its mix from " France" (see [One attention step](transformer.md#one-attention-step)), head 5, with illustrative shares, 61 % from " capital". Table o puts the 32 outputs together, so the list of " is" carries both, which the output table needs to give " Paris" first. Layer 2 repeats this with its own tables, on the lists layer 1 has already reworked, so it can build on what layer 1 brought (see [What the layers are for](transformer.md#what-the-layers-are-for)).

32 heads in each of 32 layers is a coincidence of this model: Llama 3.1 70B has 64 heads in each of its 80 layers. Nobody decides what a head looks for, and a head learns nothing itself: training adjusts tables q, k, v and o, a head uses its slice of them, and what it ends up doing is found afterwards, by measuring. Researchers have found heads that follow the previous token, and heads that find an earlier occurrence of the current token and copy what came after it (induction heads, [Olsson et al., 2022](https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads/index.html)); most are harder to name. Heads are the only place where a token reads the other tokens; feed-forward then works on each token alone (see [Feed-forward](transformer.md#feed-forward)).

#### One attention step

Inside one head, " is" compares its query with the keys of the tokens up to it, turns the scores into shares and mixes the tokens' values. Every column does the same with its own query; the diagram shrinks 128 numbers to 4. The name comes from translation models of 2014, where these shares showed which words of the source sentence the model paid attention to while writing each word ([Bahdanau et al.](https://arxiv.org/abs/1409.0473)).

<figure><img src="../../../.gitbook/assets/5.transformer_ai-transformer-one-head_v2.png" alt=""><figcaption></figcaption></figure>

* **In and out:** What goes in is q1 of " is", numbers 1 to 128 of its query (table q), already turned for its position, and k1 and v1 of the five tokens up to it (tables k and v; at use, read from the KV cache). What comes out is 128 numbers for " is" alone, useful once table o has added them to its list (D).
* **A, scores:** The query of " is" is multiplied number by number with the key of each token up to it, and the products are added: the calculation of a score in [from the final list to percentages](./#from-the-final-list-to-percentages). The angles are inside these numbers: the query and the keys were turned by their positions before (see [Position](transformer.md#position)), so the same words 500 tokens apart would give other scores. Dividing by the square root of the list size (√128 ≈ 11.3 in Llama 3.1 8B) keeps the scores small.
* **B, shares:** Softmax raises e = 2.718 to the power of each score, which makes every value positive and widens the gaps, then divides each by their total: shares of attention that add up to 100 %, how much of each token's value " is" takes. They are not chances of a word, which come only at the end of the model. " France" gets 53 %: of what head 1 brings to " is", 53 % comes from the value of " France". It is not a likeness, nor the weight of " France" in the whole text: the five shares of this head, for this token, add up to 100 %, and another head gives other shares (see Heads, C). " France" gets the most because its key agrees most with the query of " is".
* **C, the mix:** Each token's value is multiplied by its share, and the results are added column by column: number 1 of the output is 0.08 × 0.1 + 0.19 × 0.5 + 0.06 × 0.0 + 0.53 × 0.8 + 0.14 × (−0.2) = 0.50, and each other number uses its own column. The output is close to the value of " France": the list of " is" now carries information about France.
* **D, where it goes:** The outputs of the 32 heads are put one after the other: out 1's 128 numbers, then out 2's, up to out 32's, one list of 4096 numbers. Table o holds 4096 rows of 4096 numbers; its columns 1 to 128 meet out 1, 129 to 256 out 2, and so on. Each row multiplies the whole list number by number and adds the products, like a score (drawn on the right of out 32): row 1 gives the 1st number of the result, its columns 1 to 128 meeting out 1, then 129 to 256 out 2, up to out 32; row 2 gives the 2nd number, up to row 4096. That result, called here the correction (papers say the attention output), is computed for this token, while table o itself stays fixed. It is added number by number to `list_374` as it came into the layer (in layer 1, row 374 of the token table; in layer 2, the list layer 1 returned): 2.0 + 0.03 = 2.03 in the diagram. In one line: new `list_374` = `list_374` + o × (out 1 | out 2 | … | out 32), where | means "followed by". The feed-forward part then adds its own correction the same way, and the next layer starts again with its own tables. After layer 32, the final norm and the output table give one score per token of the vocabulary, turned into percentages where " Paris" is highest (as in from the final list to percentages); without attention, " is" would know nothing of " France" (see [What the layers are for](transformer.md#what-the-layers-are-for)).

At use, the [KV cache](../../inference/text-generation/kv-cache.md) keeps the keys and values of earlier tokens, so only the new token's query, key and value are computed; no query is kept, since only the new token asks. The shares are computed again, from zero, in every head of every layer.

#### Position

The attention score of [One attention step](transformer.md#one-attention-step), the query of " is" × the key of " France" in each head of each layer, compares what they say, not where their tokens are, yet the distance between two words matters. A token's **position** is its place in the text: "The" is 1, " France" 4, " is" 5 (not its token number, 9822 for " France"). Llama turns each query and key according to its position before the scores, so that a score also depends on how far apart the two tokens are: **rotary position embedding** (RoPE).

<figure><img src="../../../.gitbook/assets/6.transformer_ai-transformer-position_v2 (1).png" alt=""><figcaption></figcaption></figure>

* **A, the problem, and what turning changes:** Without turning, the same query and the same key give the same score whether " France" is just before " is" or 500 tokens back, yet some heads look for the token just before (see [Heads](transformer.md#heads)). Turned, the same pair gives 0.87 at 1 token apart and −0.50 at 500 apart in the diagram: the score now depends on the distance.
* **B, the idea:** In layer 1, " is" gets a query of 4096 numbers from table q; head 1 uses numbers 1 to 128, grouped two by two into 64 **pairs**: pair 1 is numbers 1 and 2, pair 64 numbers 127 and 128. A pair is never two tokens side by side: it is two numbers of one token's query or key, drawn as an arrow from the centre of a dial, like the x and y of a point on a map: turning needs two numbers, and one number alone cannot turn. The arrow has a length and a direction, both set by table q or k: what the token says. Turning adds an angle to that direction, position × the pair's speed: in the diagram pair 1 turns 30° per position, a speed made up so the angles are easy to see (the real pair 1 turns 57.3° per position, so " capital" at 2 gets 114.6°), and " France" (4) gets +120° and " is" (5) +150°, and pair 1 of the query of " is", (−1.12, −0.07), becomes (1.0, −0.5), the numbers of One attention step. So there are two angles: the pair's own direction, from the tables, and the angle added by its position.
* **C, why the gap changes the score:** A score multiplies each pair of the query by the same pair of the key and adds: 1.0 × 1.0 + (−0.5) × (−0.6) = 1.30 for pair 1 of " is" and " France". The closer the two arrows' directions, the larger the product. Turning added 150° to one and 120° to the other, so it moved them 30° relative to each other, from 25.6° apart to 4.4° (−25.6° + 30°): 1 token apart, the same at positions 104 and 105. With " France" at position 2, turning would move them 90° and the product would drop to 0.57. Training sets tables q and k so that a head scores high at the gaps it needs.
* **D, where it happens:** Every token turns its query and its key for its own position (in the diagram, pair 2 turns 10° per position). The query of " is" from table q, −1.12, −0.07, 1.66, 0.35, becomes 1.0, −0.5, 0.8, 1.5, and the key of " France" becomes 1.0, −0.6, 0.8, 1.2: the numbers of One attention step. A token's query is used when it reads (" France" reading " capital"), its key when it is read (" is" reading " France"); " is" reads the five tokens up to it, positions 1 to 5, each key turned for its own position. The 3.74 of One attention step, before ÷ 2, is pair 1's 1.30 plus pair 2's 0.8 × 0.8 + 1.5 × 1.2 = 2.44 (64 pairs in the model): the positions are not added to the sum, they turned the query and the key before it, in every layer. The KV cache keeps the keys already turned; values are never turned.
* **E, 64 speeds at once:** Nothing is chosen: every score adds the products of all 64 pairs. The fastest pair turns 57° per token and tells neighbours apart; the slowest needs 2.6 million tokens for a full turn and keeps track of long distances. All the speeds come from one design number in the metadata, the base: 500,000 in Llama 3.
* **F, the 64-number table:** A Llama 3.1 `.gguf` holds one more table, `rope_freqs.weight`, with one number per pair, how much slower that pair turns: 1 for the 29 fastest, 8 for the 29 slowest, in between for the 6 in the middle. Meta set this scaling when it extended the context from 8,192 tokens (Llama 3) to 131,072 (Llama 3.1). It is not learned: the converter computes it from `config.json`, so the `.gguf` holds 292 tables. The other diagrams of the wiki count the 291 tables of weights, the model itself; this one is a constant stored for convenience.
* **G, why the window stops at 131,072:** What a pair sees is the gap between two tokens. The fast pairs wrap around: many gaps give them the same angle, so alone they cannot say how far back a token is. The slow pairs can: for pair 64, slowed 8 times by F, a key 1,000 tokens back gives 0.02° and one 100,000 back 1.76°. Its angle grows with the gap, and across the longest gap of training, 131,072 tokens, it reached only 2.3°. Two tokens 1,000,000 apart give it 17.6°, the real angle and not a remainder (this pair needs 20 million tokens for one turn): an angle tables q and k never met, so the scores are no longer reliable, like an hour hand only ever seen between 12:00 and 12:05, which at 12:35 points where you never learned to read it. Hence the maximum in `config.json`, `max_position_embeddings` 131,072, past which [llama.cpp](https://github.com/ggml-org/llama.cpp) warns ("possible training context overflow"). Nothing in the tables grows with the window: a longer one needs training on longer texts, to show the model those angles and what links that far apart mean, and each token reads all those before it, so the cost grows faster than the length. Llama 3.1 was first trained on texts of 8,192 tokens, then on longer ones up to 128K in a final stage; a LLaMA used past its trained length without such a fix gets a perplexity comparable to an untrained model's ([Chen et al., 2023](https://arxiv.org/abs/2306.15595)).

#### Feed-forward

The second part of each layer works on each column alone: it reads no other token, but the list it gets already carries what attention brought in (for " is", "The capital of France"). Feed-forward adds what the model learned in training, switched on by what that list holds (a word, a fact), with three tables, gate, up and down (also called the FFN or MLP).

<figure><img src="../../../.gitbook/assets/7.transformer_ai-transformer-feed-forward_v2.png" alt=""><figcaption></figcaption></figure>

* **A, three tables:** After ffn\_norm, tables gate and up each turn the 4096 numbers into 14,336; down brings the result back to 4096, a correction added to the list that came in. Up widens the list; gate says how much of each of up's numbers passes, like a valve.
* **B, the gate and the switch:** Gate's numbers first go through a switch (SiLU) that keeps positive numbers almost as they are and shrinks negative ones toward 0; the result is multiplied number by number with up's. Where the gate is strongly positive, up's number passes; where it is negative, little does. This design is called SwiGLU.
* **C, what it holds:** Gate, up and down hold 176 million of the 218 million weights of each layer, 70 % of the whole model. Much of what the model learned seems to be stored there: researchers read these tables as memories that a pattern in the list switches on (Geva et al., 2021).
* **D, where the numbers go:** The list of " is" leaves attention as 2.03, −4.02, 4.05, 0.01… (One attention step, D), and ffn\_norm scales it. Row 1 of gate times that list, number by number, added, gives 2.00, the 1st of gate's 14,336 numbers, and row 1 of up gives 0.50; after the switch and the multiplication of B, row 1 of down times the 14,336 results gives 0.14, the 1st number of the correction, row 2 its 2nd, up to row 4096. The badges D1 to D5 mark these steps in A. Added to the list kept aside, it gives 2.17: the list that layer 2 starts from. In one line: new `list_374` = `list_374` + down × (switch(gate × n) × (up × n)), n being the list after ffn\_norm.

A **mixture of experts** is another way to build this part: many smaller sets of gate, up and down tables, the experts, and a router that sends each token to a few of them (see [Mixture of experts](mixture-of-experts.md)).

#### A word in several pieces

A word cut into several tokens enters the model as several columns. The token table holds lists for pieces, not for words, so the model puts the word back together at its last piece, where the tokens after it read it.

<figure><img src="../../../.gitbook/assets/8.transformer_ai-transformer-word-pieces_v1.png" alt=""><figcaption></figcaption></figure>

* **One piece at a time:** In "Strawberry", cut into "Str", "aw" and "berry" (see [Tokens](../../inference/text-generation/tokens.md)), the column of "Str" never reads "aw". In a prompt the pieces are computed together, each reading only the pieces before it; in an answer, "Str" is written before "aw" exists.
* **Gathered at the last piece:** "Strawberry" is not a token, so there is no column for the whole word; "berry" is the only column that has read all three pieces, since no column reads a later one. Every layer has both parts, but each does this job in different layers: in the first few, the attention of "berry" puts a large share on "Str" and "aw"; in the early and middle layers, the feed-forward part adds what the model knows about the whole word. Researchers measured this in Llama 2 7B and similar models.
* **Known or invented:** An invented word such as "koweight" ("k", "owe", "ight") is gathered the same way; the feed-forward parts still run, but find no stored word to add, and the list keeps what its pieces and the context say. A typo sits in between: its pieces differ from the real word's, yet models often recover it from the pieces and the context (GPT-4 restored sentences whose letters were shuffled in every word, [Cao et al., 2023](https://arxiv.org/abs/2311.18805)). A probe, a small classifier the researchers trained on these lists, tells real words from invented ones 89 % of the time at layer 13 of Llama 2 7B.

The model never sees letters, only pieces and what its layers rebuild from them, which is why counting the r's of "Strawberry" is hard for it (see **Limits**).

#### Other designs

Most current models change the details of this plan, not the plan itself.

* **Encoder-decoder:** The 2017 design, built for translation, had two stacks: an **encoder** that read the sentence to translate without a mask, and a **decoder** that wrote the translation with it. Llama and GPT models keep only the decoder (**decoder-only**); models built only to read text, such as BERT, keep the encoder. In the paper's figure, reused in many articles, the output table is labelled Linear.
* **Experts:** The feed-forward part split into experts, a few used per token (Mixture of experts).
* **Lighter attention:** Keys and values stored compressed (MLA, DeepSeek), attention limited to recent tokens in some layers (gpt-oss), or replaced in most layers by a mechanism with a fixed-size state (36 of the 48 layers of Qwen3.5-122B-A10B). All shrink the KV cache (see [what labs do about it](../../inference/text-generation/kv-cache.md#what-labs-do-about-it)).
* **Pictures:** Image models such as Flux and Stable Diffusion 3 use a transformer in place of the older U-Net, working on patches of the latent (a diffusion transformer, DiT; see [Image & video](../../inference/image-and-video.md)). The vision encoder of a multimodal model is a transformer too (see **Multimodal models**).

#### **Resources**

* Papers: [Attention Is All You Need](https://arxiv.org/abs/1706.03762) (Vaswani et al., 2017) | [Bahdanau et al., Neural Machine Translation by Jointly Learning to Align and Translate](https://arxiv.org/abs/1409.0473) (2014: attention, added to translation models) | [Shazeer, GLU Variants Improve Transformer](https://arxiv.org/abs/2002.05202) (2020: SwiGLU, the gate of the feed-forward part) | [GQA](https://arxiv.org/abs/2305.13245) (Ainslie et al., 2023) | [RoFormer](https://arxiv.org/abs/2104.09864) (Su et al., 2021: rotary position embedding) | [Chen et al., Extending Context Window of Large Language Models via Positional Interpolation](https://arxiv.org/abs/2306.15595) (2023: what happens past the trained length, and slowing the turns) | [Scalable Diffusion Models with Transformers](https://arxiv.org/abs/2212.09748) (Peebles and Xie: DiT)
* Llama 3.1: [The Llama 3 Herd of Models](https://arxiv.org/abs/2407.21783) (32 heads, 8 key-value heads, RoPE base 500,000, context raised from 8K to 128K in six stages) | [llama.cpp, Llama converter](https://github.com/ggml-org/llama.cpp/blob/master/conversion/llama.py) (`rope_freqs` computed from `rope_scaling`)
* Inside the layers: [Geva et al., Transformer Feed-Forward Layers Are Key-Value Memories](https://arxiv.org/abs/2012.14913) (EMNLP 2021: what feed-forward parts store) | [Kaplan et al., From Tokens to Words](https://arxiv.org/abs/2410.05864) (words rebuilt at their last piece; probe at layer 13) | [Olsson et al., In-context Learning and Induction Heads](https://transformer-circuits.pub/2022/in-context-learning-and-induction-heads/index.html) (Anthropic, 2022)
* Videos: [3Blue1Brown, Transformers, the tech behind LLMs](https://www.3blue1brown.com/lessons/gpt) (chapter 5: the whole model, visually) | [3Blue1Brown, Attention in transformers](https://www.youtube.com/watch?v=eMlx5fFNoYc) (chapter 6: queries, keys and values, visually) | [Andrej Karpathy, Let's build GPT](https://www.youtube.com/watch?v=kCc8FmEb1nY) (2023: a small transformer written in code, two hours)
