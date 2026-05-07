dimension:

 $$ C^{a}=H\cdot W^{a K V},\quad C^{b}=H\cdot W^{b K V}, $$ 

 $$ Z^{a}=H\cdot W^{a Z},\qquad Z^{b}=H\cdot W^{b Z}, $$ 

where  $ W^{aKV} $,  $ W^{bKV} $,  $ W^{aZ} $,  $ W^{bZ} \in \mathbb{R}^{d \times c} $ are trainable parameters. Next, each  $ m $ KV entries in  $ C^a $ and  $ C^b $ will be compressed into one entry according to their compression weights and learnable positional biases  $ B^a $,  $ B^b \in \mathbb{R}^{m \times c} $, producing  $ C^{\text{Comp}} \in \mathbb{R}^\frac{n}{m} \times c $. Each compressed entry  $ C_i^{\text{Comp}} \in \mathbb{R}^c $ is computed by

 $$ [S_{m i:m(i+1)-1}^{a};S_{m(i-1):m i-1}^{b}]=\operatorname{S o f t m a x}_{\operatorname{r o w}}\big([Z_{m i:m(i+1)-1}^{a}+B^{a};Z_{m(i-1):m i-1}^{b}+B^{b}]\big), $$ 

 $$ C_{i}^{Comp}=\sum_{j=mi}^{m(i+1)-1}S_{j}^{a}\odot C_{j}^{a}+\sum_{j=m(i-1)}^{mi-1}S_{j}^{b}\odot C_{j}^{b}, $$ 

where $\odot$ denotes the Hadamard product; Softmax$_{row}(\cdot)$ denotes the softmax operation along the row dimension, which performs normalization across the total of $2m$ elements from both $Z^a$ and $Z^b$. When $i = 0$, $Z_{m(i-1):mi-1}^b$ is padded with negative infinity and $C_{m(i-1):mi-1}^b$ is padded with zeros. Note that each $C_i^{Comp}$ is derived from $2m$ KV entries, but the indexes of $C^b$ used for $C_i^{Comp}$ and the indexes of $C^a$ used for $C_{i-1}^{Comp}$ are overlapped. Therefore, CSA in fact compresses the sequence length to $\frac{1}{m}$ times.

Lightning Indexer for Sparse Selection. After obtaining the compressed KV entries  $ C^{Comp} $, CSA applies the DSA strategy to select top-k compressed KV entries for core attention. First, CSA performs the same compression operation used for  $ C^{Comp} $ to get compressed indexer keys  $ K^{IComp} \in \mathbb{R}^{\frac{n}{m} \times c^{l}} $, where  $ c^{l} $ is the indexer head dimension. Then, for a query token  $ t $, we produce the indexer queries  $ \{q_{t,1}^{l}, q_{t,2}^{l}, \ldots, q_{t,n_{t}^{l}}^{l}\} $ in a low-rank manner:

 $$ \mathbf{c}_{t}^{Q}=\mathbf{h}_{t}\cdot W^{D Q}, $$ 

 $$ [\mathbf{q}_{t,1}^{I};\mathbf{q}_{t,2}^{I};...;\mathbf{q}_{t,n_{h}^{I}}^{I}]=\mathbf{q}_{t}^{I}=\mathbf{c}_{t}^{Q}\cdot W^{I U Q}, $$ 

where  $ \mathbf{h}_t \in \mathbb{R}^d $ is the input hidden state of the query token  $ t $;  $ \mathbf{c}_t^Q \in \mathbb{R}^d_c $ is the compressed latent vector for queries;  $ d_c $ denotes the query compression dimension;  $ n_h^l $ denotes the number of indexer query heads;  $ W^{DQ} \in \mathbb{R}^{d \times d_c} $ and  $ W^{IUQ} \in \mathbb{R}^{d_c \times c_l^{n_h}} $ are the down-projection and up-projection matrices for indexer queries, respectively. Next, the index score  $ I_{t,s} \in \mathbb{R} $ between the query token  $ t $ and a preceding compressed block  $ s $ ( $ s < \text{Floor}(\frac{t}{m}) $) is computed by

 $$ [w_{t,1}^{I};w_{t,2}^{I};...;w_{t,n_{h}^{I}}^{I}]=\mathbf{w}_{t}^{I}=\mathbf{h}_{t}\cdot W^{w}, $$ 

 $$ I_{t,s}=\sum_{h=1}^{n_{h}^{I}}w_{t,h}^{I}\cdot\mathrm{R e L U}\left(\mathbf{q}_{t,h}^{I}\cdot K_{s}^{\mathrm{I C o m p}}\right), $$ 

where  $ W^{w} \in \mathbb{R}^{d \times n_h} $ is a learnable matrix;  $ w_{t,h}^I \in \mathbb{R} $ is the weight of the  $ h $-th indexer head. For a query token  $ t $, given its index scores  $ I_{t,:} $, we employ a top- $ k $ selector to selectively retain a subset of compressed KV entries  $ C_t^{SprsComp} $ for subsequent core attention:

 $$ C_{t}^{S p r s C o m p}=\left\{C_{s}^{C o m p}\;\middle|\;I_{t,s}\in\mathrm{T o p-k}(I_{t,:})\right\}. $$ 