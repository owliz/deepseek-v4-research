## Contents

1 Introduction 4  
  
2 Architecture 6  
2.1 Designs Inherited from DeepSeek-V3 7  
2.2 Manifold-Constrained Hyper-Connections 7  
2.3 Hybrid Attention with CSA and HCA 9  
2.3.1 Compressed Sparse Attention 9  
2.3.2 Heavily Compressed Attention 11  
2.3.3 Other Details 12  
2.3.4 Efficiency Discussion 13  
2.4 Muon Optimizer 14  
  
3 General Infrastructures 15  
3.1 Fine-Grained Communication-Computation Overlap in Expert Parallelism 15  
3.2 Flexible and Efficient Kernel Development with TileLang 16  
3.3 High-Performance Batch-Invariant and Deterministic Kernel Libraries 18  
3.4 Training Framework 19  
3.4.1 Efficient Implementation of Muon 19  
3.4.2 Cost-Effective and Memory-Efficient Implementation of mHC 20  
3.4.3 Contextual Parallelism for Long-Context Attention 20  
3.4.4 Extended Automatic Differentiation for Flexible Activation Checkpointing 21  
3.5 Inference Framework 21  
3.5.1 KV Cache Structure and Management 21  
3.5.2 On-Disk KV Cache Storage 23  
  
4 Pre-Training 24  
4.1 Data Construction 24  
4.2 Pre-Training Setups 24  
4.2.1 Model Setups 24  
4.2.2 Training Setups 25  
4.2.3 Mitigating Training Instability 26  
4.3 Evaluations 27  
4.3.1 Evaluation Benchmarks 27  
4.3.2 Evaluation Results 27