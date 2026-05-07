Algorithm 1 Muon Optimizer for DeepSeek-V4

Require: Learning rate  $ \eta $, momentum  $ \mu $, weight decay  $ \lambda $, update rescaling factor  $ \gamma $

1: for each training step t do

2:     for each logically independent weight  $ W \in \mathbb{R}^{n \times m} $ do

3:          $ G_t = \nabla_W \mathcal{L}_t(W_{t-1}) $

4:          $ M_t = \mu M_{t-1} + G_t $

5:          $ O_t' = \text{HybridNewtonSchulz}(\mu M_t + G_t) $

6:          $ O_t = O_t' \cdot \sqrt{\max(n, m)} \cdot \gamma $

7:          $ W_t = W_{t-1} \cdot (1 - \eta \lambda) - \eta O_t $

8:     end for

9: end for

Moreover, even when compared with DeepSeek-V3.2 (DeepSeek-AI, 2025) — already an efficient baseline — DeepSeek-V4 series still exhibits substantial advantages in efficiency. A comparison of their inference FLOPs and KV cache size is provided in the right part of Figure 1.

#### 2.4. Muon Optimizer

We employ the Muon (Jordan et al., 2024; Liu et al., 2025) optimizer for the majority of modules in DeepSeek-V4 series due to its faster convergence and improved training stability. The full algorithm of our Muon optimization is summarized in Algorithm 1.

Basic Configurations. We maintain the AdamW (Loshchilov and Hutter, 2017) optimizer for the embedding module, the prediction head module, the static biases and gating factors of mHC modules, and the weights of all RMSNorm modules. All other modules are updated with Muon. Following Liu et al. (2025), we also apply weight decay to Muon parameters, use the Nesterov (Jordan et al., 2024; Nesterov, 1983) trick, and rescale the Root Mean Square (RMS) of the update matrix for reutilization of our AdamW hyper-parameters. Different from them, we use hybrid Newton-Schulz iterations for orthogonalization.

Hybrid Newton-Schulz Iterations. For a given matrix  $ M $, let its Singular Value Decomposition (SVD) be  $ M = U \Sigma V^T $. The Newton-Schulz iterations aim to approximately orthogonalize  $ M $ to be  $ UV^T $. Usually,  $ M $ will be first normalized as  $ M_0 = M / \|M\|_F $ to ensure its maximum singular value does not exceed 1. Then, each Newton-Schulz iteration performs the following operation:

 $$ M_{k}=a M_{k-1}+b\big(M_{k-1}M_{k-1}^{T}\big)M_{k-1}+c\big(M_{k-1}M_{k-1}^{T}\big)^{2}M_{k-1}. $$ 

Our hybrid Newton-Schulz performs 10 iterations over two distinct stages. During the first 8 steps, we use coefficients  $ (a, b, c) = (3.4445, -4.7750, 2.0315) $ to drive rapid convergence, bringing the singular values close to 1. In the final 2 steps, we switch to coefficients  $ (a, b, c) = (2, -1.5, 0.5) $, which stabilize the singular values precisely at 1.

Avoiding Exploding Attention Logits. The attention architecture of DeepSeek-V4 series allows us to directly apply RMSNorm on the attention queries and KV entries, which effectively prevents attention logits from exploding. Consequently, we do not employ the QK-Clip technique (Liu et al., 2025) in our Muon optimizer.