compression. Let  $ H \in \mathbb{R}^{n \times d} $ be a sequence of input hidden states, HCA first computes the original KV entries  $ C \in \mathbb{R}^{n \times c} $ and their corresponding compression weights  $ Z \in \mathbb{R}^{n \times c} $:

 $$ C=H\cdot W^{K V}, $$ 

 $$ Z=H\cdot W^{Z}, $$ 

where  $ W^{KV}, W^Z \in \mathbb{R}^{d \times c} $ are trainable parameters. Next, each  $ m' $ KV entries in  $ C $ will be compressed into one according to the compression weights and learnable positional biases  $ B \in \mathbb{R}^{m' \times c} $, producing  $ C^{\text{Comp}} \in \mathbb{R}^{\frac{n}{m' \times c}} $. Each compressed entry  $ C_i^{\text{Comp}} \in \mathbb{R}^c $ is computed by

 $$ S_{m^{\prime}i:m^{\prime}(i+1)-1}=Softmax_{row}\left(Z_{m^{\prime}i:m^{\prime}(i+1)-1}+B\right), $$ 

 $$ C_{i}^{\mathrm{C o m p}}=\sum_{j=m^{\prime}i}^{m^{\prime}(i+1)-1}S_{j}\odot C_{j}. $$ 

Through this compression operation, HCA compresses the sequence length to  $ \frac{1}{m'} $ times.

Shared Key-Value MQA and Grouped Output Projection. HCA also employs the shared KV MQA and grouped output projection strategies as CSA does. After the KV compression, for a query token t, HCA first produces attention queries  $ \{\mathbf{q}_{t,1}; \mathbf{q}_{t,2}; \ldots; \mathbf{q}_{t,n_{h}}\} $ in a low-rank manner:

 $$ \mathbf{c}_{t}^{Q}=\mathbf{h}_{t}\cdot W^{D Q}, $$ 

 $$ [\mathbf{q}_{t,1};\mathbf{q}_{t,2};...;\mathbf{q}_{t,n_{h}}]=\mathbf{q}_{t}=\mathbf{c}_{t}^{Q}\cdot W^{U Q}, $$ 

where  $ \mathbf{h}_t \in \mathbb{R}^d $ is the input hidden state of the query token  $ t $;  $ n_h $ denotes the number of query heads;  $ W^{DQ} \in \mathbb{R}^{d \times d_c} $ and  $ W^{UQ} \in \mathbb{R}^{d_c \times c n_h} $ are the down-projection and up-projection matrices for queries, respectively. Next, we perform MQA on  $ \{\mathbf{q}_{t,i}\} $ and  $ C^{\text{Comp}} $:

 $$ \mathbf{o}_{t,i}=CoreAttn\left(\mathbf{q u e r y}=\mathbf{q}_{t,i},\mathbf{k e y}=C^{Comp},\mathbf{v a l u e}=C^{Comp}\right), $$ 

where  $ \mathbf{o}_{t,i} \in \mathbb{R}^c $ is the core attention output of the  $ i $-th head at the  $ t $-th token. Next, as CSA does, HCA splits  $ n_h $ outputs into  $ g $ groups, and for each group of output  $ \mathbf{o}_{t,i}^G \in \mathbb{R}^{c^{\frac{n_h}{g}}} $, HCA projects it to a  $ d_g $-dimensional intermediate output  $ \mathbf{o}_{t,i}^G' \in \mathbb{R}^{d_g} $, where  $ d_g < c^{\frac{n_h}{g}} $. Finally, HCA projects the intermediate output  $ [\mathbf{o}_{t,1}^G'; \mathbf{o}_{t,2}^G'; \ldots; \mathbf{o}_{t,g}^G'] \in \mathbb{R}^{d_g} $ to the final attention output  $ \hat{\mathbf{o}}_t \in \mathbb{R}^d $.

##### 2.3.3. Other Details

In addition to the core architectures of CSA and HCA described above, our hybrid attention incorporates several other techniques. For writing clarity, we omit these additional techniques from the above introduction and will briefly describe them in this subsection. Also, this subsection focuses only on the core ideas of them and may omit some tiny details for simplicity. We encourage the readers to refer to our open-source implementation for unambiguous details.

Query and Key-Value Entry Normalization. For both CSA and HCA, we perform an additional RMSNorm operation on each head of the queries and the only head of the compressed KV entries, just before the core attention operation. This normalization avoids exploding attention logits and may improve training stability.