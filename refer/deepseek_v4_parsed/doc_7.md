width does not influence the design of the inner layers. HC decouples the residual width from the actual hidden size, offering a complementary scaling axis with minimal computational overhead, as  $ n_{hc} $ is typically much smaller than the hidden size d. However, even though HC has demonstrated potential in improving model performance, we find that the training will frequently exhibit numerical instability when stacking multiple layers, which hinders the scaling of HC.

Manifold-Constrained Residual Mapping. The core innovation of mHC is to constrain the residual mapping matrix  $ B_{l} $ to the manifold of doubly stochastic matrices (the Birkhoff polytope) M, and thus enhance the stability of signal propagation across layers:

 $$ B_{l}\in\mathcal{M}:=\{M\in\mathbb{R}^{n\times n}\mid M\mathbf{1}_{n}=\mathbf{1}_{n},\mathbf{1}_{n}^{T}M=\mathbf{1}_{n}^{T},M\geqslant0\}. $$ 

This constraint ensures that the spectral norm of the mapping matrix  $ \|B_l\|_2 $ is bounded by 1, so the residual transformation is non-expansive, which increases the numerical stability during both the forward pass and backpropagation. Besides, the set  $ M $ is closed under multiplication, which guarantees stability in the scenarios of deep stacks of mHC. In addition, the input transformation  $ A_l $ and output transformation  $ C_l $ are also constrained to be non-negative and bounded via a Sigmoid function to avoid the risk of signal cancellation.

Dynamic Parameterization. The parameters of three linear mappings are dynamically generated, which are decomposed into a dynamic (input-dependent) component and a static (input-independent) component. Given the input  $ X_l \in \mathbb{R}^{n_h \times d} $, it is first flattened and normalized:  $ \hat{X}_l = \text{RMSNorm}(\text{vec}(X_l)) \in \mathbb{R}^{1 \times n_h \times d} $. Then, we follow the conventional HC to generate the unconstrained raw parameters  $ \hat{A}_l \in \mathbb{R}^{1 \times n_h} $,  $ \hat{B}_l \in \mathbb{R}^{n_h \times n_h} $, and  $ \tilde{C}_l \in \mathbb{R}^{n_h \times 1} $:

 $$ \tilde{A}_{l}=\alpha_{l}^{\mathrm{p r e}}\cdot(\hat{X}_{l}W_{l}^{\mathrm{p r e}})+S_{l}^{\mathrm{p r e}}, $$ 

 $$ \tilde{B}_{l}=\alpha_{l}^{\mathrm{r e s}}\cdot\mathrm{M a t}(\hat{X}_{l}W_{l}^{\mathrm{r e s}})+S_{l}^{\mathrm{r e s}}, $$ 

 $$ \tilde{C}_{l}=\alpha_{l}^{\mathrm{p o s t}}\cdot(\hat{X}_{l}W_{l}^{\mathrm{p o s t}})^{T}+S_{l}^{\mathrm{p o s t}}, $$ 

where  $ W_l^{\text{pre}}, W_l^{\text{post}} \in \mathbb{R}^{n_{\text{hc}}d \times n_{\text{hc}}} $ and  $ W_l^{\text{res}} \in \mathbb{R}^{n_{\text{hc}}d \times n_{\text{hc}}^2} $ are learnable parameters for generating the dynamic components; Mat( $ \cdot $) reshapes a vector of size  $ 1 \times n_{\text{hc}}^2 $ into a matrix of size  $ n_{\text{hc}} \times n_{\text{hc}} $;  $ S_l^{\text{pre}} \in \mathbb{R}^{1 \times n_{\text{hc}}}, S_l^{\text{post}} \in \mathbb{R}^{n_{\text{hc}} \times 1} $, and  $ S_l^{\text{res}} \in \mathbb{R}^{n_{\text{hc}} \times n_{\text{hc}}} $ are learnable static biases; and  $ \alpha_l^{\text{pre}}, \alpha_l^{\text{res}}, \alpha_l^{\text{post}} \in \mathbb{R} $ are learnable gating factors initialized to small values.

Applying Parameter Constraints. After obtaining the unconstrained raw parameters  $ \tilde{A}_l $,  $ \tilde{B}_l $,  $ \tilde{C}_l $, we then apply constraints described earlier to them to enhance the numerical stability. To be specific, for the input and output mappings, we employ a Sigmoid function  $ \sigma(\cdot) $ to ensure their non-negativity and boundedness:

 $$ A_{l}=\sigma(\tilde{A}_{l}), $$ 

 $$ C_{l}=2\sigma(\tilde{C}_{l}). $$ 

As for the residual mapping  $ \tilde{B}_l $, we project it onto the manifold of doubly stochastic matrices  $ M $. This is achieved by the Sinkhorn-Knopp algorithm, which first applies an exponential function to  $ \tilde{B}_l $ to ensure positivity, getting  $ M^{(0)} = \exp(\tilde{B}_l) $, and then iteratively performs column and row normalization;

 $$ M^{(t)}=\mathcal{T}_{r}(\mathcal{T}_{c}(M^{(t-1)})), $$ 

where  $ \mathcal{T}_r $ and  $ \mathcal{T}_c $ denote row and column normalization, respectively. This iteration converges to a constrained doubly stochastic matrix  $ B_l = M^{(t_{\max})} $. We choose  $ t_{\max} = 20 $ as a practical value.