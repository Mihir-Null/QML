1. **Title: Implementing a Hybrid Transitional Optimizer to mitigate barren plateaus in QAOA for Max-Cut**  
2. **Introduction**  
   1. Motivation: The Barren Plateau Problem presents a critical challenge in the optimization of variational quantum circuits, a cornerstone of quantum machine learning (QML). As circuits grow in complexity with additional qubits and layers, the cost function landscape flattens, exponentially reducing the likelihood of identifying optimal parameters via gradient descent. This happens due to the concentration of measure phenomenon for large parametrized unitaries. This is prevalent in utility scale implementations for circuits such as QAOA when implementing solving problems such as Max-Cut for utility scale graphs of large numbers of nodes. To mitigate this effect it is common to use non gradient optimizers when optimizing using large unitaries such as QAOA  
   2. Our algorithm to test on is QAOA for Max Cut, which refers to using the Quantum Approximate Optimization Algorithm (QAOA) to solve the "Max Cut" problem, a combinatorial optimization challenge where the goal is to partition the nodes of a graph into two sets in a way that maximizes the number of edges connecting nodes from different sets, essentially "cutting" as many edges as possible across the partition; essentially, it's a quantum computing approach to finding the best possible division of a graph into two groups based on edge connections.  
   3. However This comes at the cost of trading off the increased granularity (and generally) speed that comes with gradient optimizers in the presence of viable gradients. Non gradient optimizers are less susceptible to the barren plateau but are generally slower and offer less precision  
   4. Objectives: Our project aims to address the Barren Plateau Problem by exploring and implementing various quantum machine learning optimizers. In particular creating a hybrid optimizer that utilizes a non gradient optimizer to find a set of good initial parameters that are then fed into a gradient based optimizer to leverage the benefits of both  
3. **Graph and Encoding**  
   1. Data Description:   
      The dataset is a graph with 16 nodes and approximately 60 edges. Nodes, which are represented as consecutive increasing integers starting at 0, represent individual entities in the graph. The edges represent connections between nodes. Edges consist of two endpoints, which are the nodes that the edge connects, and weight, which is the significance of the connection. In the code, an edge is represented by a tuple (u, v, weight), where u and v are the node numbers and weight is the edge’s weight. An example of an edge tuple is (0, 1, 1.0), which represents an edge connecting nodes 0 and 1, with a weight of 1.0. There is a probability of .5 added to the edges generated, so while the total number of possible edges is n(n-1)/2, because the probability is .5, roughly half that number of edges will be generated. For 16 nodes, that value is 60\.  
        
      Data’s purpose:  
        
      The graph, which is our data, is used to model the Max-Cut problem. The goal of this problem is to split the nodes into two subsets in a manner that maximizes the sum of the weights of the edges between the subsets.  
        
   2. Preprocessing Steps:

      The graph is first represented as an adjacency structure (data structure representing a graph) through the Rustworkx library, as was described in the data’s description. This prepares the data to be encoded into a quantum Hamiltonian.   
        
      To encode the Hamiltonian, the graph’s edges and weights are converted into Pauli-Z operators. For each edge in the graph, a string of identity operators (I) are created for all of the nodes (i.e. for data with five nodes, it looks like this: \[“I”, “I”, “I”, “I”, “I”\]) , and the nodes the edge connects will be replaced by Z (i.e. for an edge tuple (0, 1, 1.0), the string looks like: \[“Z”, “Z”, “I”, “I”, “I”\] ). Then the edge’s weight will be associated with the string and this tuple will finally be appended to the Hamiltonian list.  (i.e. for the edge in the previous example, the tuple would look like (“ZZIII”, 1.0). When this is done for each edge, the data will be a list of tuples, and the list represents the quantum cost Hamiltonian.

      The quantum Hamiltonian is used to define the quantum circuit along with initial parameters. The circuit is a QAOA ansatz, which will be executed to collect data. The QAOA ansatz is, as explained before, large and nontrivial for large graphs, for example here is the corresponding ansatz for the graph below  
      ![][image1]  
        
   3. Data Visualization

      For n \= 16, which was used for our data, a possible graph look like:  
      ![][image2]

4. **Methods**  
   1. This project was implemented using Qiskit, an open-source quantum computing framework, for constructing, simulating, and optimizing the quantum circuits. Additional tools and libraries included:  
      Rustworkx: For graph generation and manipulation.  
      Qiskit Runtime: For managing hybrid quantum-classical workflows and leveraging IBM's simulated quantum backends (FakeWashingtonV2) for circuit execution.  
      Python Libraries: NumPy, SciPy, and Matplotlib were used for numerical computations and visualization.  
      **Model Architecture**  
      The quantum circuit architecture consists of two main components:  
      Graph Encoding: A graph with n \= 16 nodes and randomly generated edges was encoded into a cost Hamiltonian representing the Max-Cut problem.  
      **QAOA Circuit**:  
      A QAOA Ansatz with p \= 2 layers was used. This was transpiled to a simulated version of the IBM washington quantum computer.  
      **Loss function:**  
      The cost is computed by simply taking the expectation value of the hamiltonian which is done by measuring out the n bit string Z=Z₁Z₂… Zn , Zᴋ∈ {0,1}ⁿ and m clauses and the aim is to maximize a given objective/cost function C=∑Cᴋ(z) where the summation goes from ᴋ=1 to m and Cᴍ ∈{0,1}  
      ![][image3]  
      And Cᴊᴋ|z\> \= 1/2(-gⱼ⊗ gᴋ \+ I)|z\> (for j and k being any two nodes/qubits)  
      such that if there is a cut then gⱼ⊗ gᴋ \=-I and zj and zk are both |0\> or |1\> if there is no cut then we have gⱼ⊗ gᴋ \= I , they belong to different qubits. Thus we get either |z\> or 0 correspondingly.  
      Essentially we sum up the number of edges “cut” (any two edges with nodes in opposite partitions)  
      We then optimize the circuit using each optimizer and postprocess  
   2. **Individual optimizers and postprocessing**  
      We tested various optimizers to find that COBYLA and SPSA work best experimentally in the categories of non gradient and gradient based optimizers respectively.  
      We first test each optimizer individually before testing our hybrid optimizer. The workflow for testing each optimizer is as follows:  
* We first optimize the circuit and obtain the result, plotting the current loss as corresponding to each call to the loss functions  
* We then create the optimized circuit with parameters gamma and beta (vectors with the depth of the circuit) and sample the most frequent bit strings, these bit strings represent the optimal partition of the graph  
* As these bit strings are long and have a complicated distribution we select the most frequently sampled one as our solution and plot it on the graph by coloring the nodes according to their partitions  
* ![][image4]  
* We then determine the “value” of the cut to understand how many edges were actually cut and understand the final efficacy of the algorithm

  

  3. **Hybrid Optimizer Design**  
     A hybrid optimization approach was employed to overcome the barren plateau problem, which occurs in quantum machine learning when gradients vanish in flat regions of the cost landscape.  
     **COBYLA (Constrained Optimization by Linear Approximations):**  
     A non-gradient-based optimizer used for broad exploration.  
     Escaped barren plateaus by approximating the cost landscape through linear approximation and iteratively reducing the loss function.  
     Effective for quickly identifying promising regions but lacks the precision to fine-tune parameters in highly curved regions.  
     **SPSA (Simultaneous Perturbation Stochastic Approximation):**  
     A gradient-based optimizer used for fine-grained refinement.  
     Efficiently estimated gradients through simultaneous random perturbations and updated parameters to converge at a local minimum.  
     Struggles in barren plateaus where gradients are effectively zero but excels at precise parameter optimization in viable regions.  
     **Switching Mechanism:**  
     Optimization began with COBYLA to explore the parameter space and escape flat regions.  
     Once the loss was reduced to 1/e of the initial value, the optimizer switched to SPSA for fine-tuning.  
     **The hybrid optimizer:**  
     Exploration Phase (COBYLA): Escaped barren plateaus and reduced loss significantly.  
     Exploitation Phase (SPSA): Refined the parameters further, leveraging gradients to achieve convergence.  
     Loss values were recorded at each iteration to analyze performance.


  

5. **Results**

**COBYLA Only** 
 
**![][image5]**  
Initial Performance: Achieved a steep reduction in cost during the first 10 iterations, dropping from approximately 3.5 to near 0\.  
Plateau: The optimizer plateaued after around 15 iterations, with minimal further improvement. The final cost value stabilized around \-0.8 with a cut value of 36\.  
Observation: COBYLA was effective at escaping the barren plateau but lacked precision in fine-tuning, resulting in suboptimal convergence.

**SPSA Only**

**![][image6]**

Initial Phase: The optimizer struggled in a clear barren plateau for the first 50-100 iterations, with the cost fluctuating around 0–5.0.  
Gradual Improvement: After finding viable gradients, the cost steadily decreased, reaching a value of approximately \-2 after 400 iterations with a higher cut value.  
Observation: SPSA achieved a lower final cost than COBYLA but required significantly more iterations due to the barren plateau

**Hybrid Optimizer**

**![][image7]**

The hybrid optimizer combined the strengths of both methods, achieving the steep initial decline of COBYLA and the precise fine-tuning of SPSA. COBYLA ran for 35 iterations. It reached its lowest final cost of almost \-1.7

We find that the hybrid optimizer achieves a minimum cost lower than COBYLA, but at a larger number of iterations. However, it takes fewer iterations than just the SPSA optimizer, but does not achieve as low a minimum cost. We can also see that the loss jumps as soon as spsa is switched to, suggesting that noise or an unviable gradient lead to spsa actually increasing the loss

While this result is a decent test of hybrid optimization, barren plateau problems are more prevalent on utility scale graphs of 100 or more nodes and testing on several randomly generated graphs. This will require a large number of consecutive jobs on an actual quantum computer as testing utility scale problems via simulation is unviable even on HPCs. Additionally it appears that noise has significantly detracted from the performance of SPSA which points to the efficacy error mitigation techniques may have on improving results.

6. **Conclusion**

   In conclusion, we attempted to tackle the Barren Plateau Problem, a significant challenge in optimizing variational quantum circuits, by creating a hybrid non-gradient-gradient optimizer. We tested our optimizer by running it on an implementation of QAOA for max-cut which is known to be prone to barren plateaus at high numbers of qubits and layers. While we see conflicting initial results, more testing is warranted.

7. **References**

* Ceroni, Jack. “Intro to QAOA.” *PennyLane Demos*, Xanadu, 18 Nov. 2020, pennylane.ai/qml/demos/tutorial\_qaoa\_intro.   
* Lockwood, Owen. “An Empirical Review of Optimization Techniques for Quantum Variational Circuits.” *ArXiv.org*, 9 Feb. 2022, [arxiv.org/abs/2202.01389](http://arxiv.org/abs/2202.01389).  
*    “Qiskit Algorithms (Qiskit\_algorithms) \- Qiskit Algorithms 0.3.1.” *Qiskit*, 2017, [qiskit-community.github.io/qiskit-algorithms/apidocs/qiskit\_algorithms.html](http://qiskit-community.github.io/qiskit-algorithms/apidocs/qiskit_algorithms.html).  
* Simon Blanke. “GitHub \-Gradient-Free-Optimizers.” *GitHub*, 14 Aug. 2024, github.com/SimonBlanke/Gradient-Free-Optimizers. 

[image1]: ./images/image1.png
[image2]: ./images/image2.png
[image3]: ./images/image3.png
[image4]: ./images/image4.png
[image5]: ./images/image5.png
[image6]: ./images/image6.png
[image7]: ./images/image7.png
