# Solution Sets

Everything in Omega operates on solutions sets.

- $\Omega$ denotes the set of all solution sets.
- $\Sigma$ denotes an individual solution set.
- $\Sigma_i=S$ denotes a solution.
- $\Sigma_ik=S_k=b$ represents a binding.

$$
\Sigma=\left\{\begin{matrix}
    \left\{\begin{matrix}
        \left("X",1\right)\\
        \left("Y",1\right)
    \end{matrix}\right\} \\
    \left\{\begin{matrix}
        \left("X",1\right)\\
        \left("Y",2\right)
    \end{matrix}\right\}
\end{matrix}\right\}\\
$$

$$
\Sigma_{0}=\left\{ \begin{matrix}
    \left("X",1\right)\\
    \left("Y",1\right)
\end{matrix}\right\}
\\
\Sigma_{1}=\left\{ \begin{matrix}
    \left("X",1\right)\\
    \left("Y",2\right)
\end{matrix}\right\}
$$

$$
\Sigma_{0"X"}=1 \\
\Sigma_{1"Y"}=2
$$

In JSON, solutions are represented as any valid json object.

```json
{ "X": 1, "Y": 2 }
```

## Bindings

Solutions are comprised of bindings. Every key may only have one value.

## Laws

Symbol distinctness law. Every symbol mapping in a solution with the same symbol will have the same value.

$$
\forall \langle k_{i}, v_{i} \rangle \langle k_{j}, v_{j} \rangle \in S \ (k_{i} \neq k_{j})
$$

### Conjunction Of Bindings

$
b\cap b=b  \\
\left\langle k,x\right\rangle \cap\left\langle k,y\right\rangle =f \\
b\cap f=f
$

### Disjunction Of Bindings

$
b\cup b=b \\
\left\langle k,x\right\rangle \cup\left\langle k,y\right\rangle =\left\{ \begin{matrix}\left\{ \left\langle k,x\right\rangle \right\} \\
\left\{ \left\langle k,y\right\rangle \right\} 
\end{matrix}\right\} \\
b\cup f=b
$

## Conjunction Examples
$
\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",1\right\rangle 
\end{matrix}\right\} \cap\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",2\right\rangle 
\end{matrix}\right\} =f \\
\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",1\right\rangle 
\end{matrix}\right\} \cap\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",1\right\rangle 
\end{matrix}\right\} =\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",1\right\rangle 
\end{matrix}\right\} \\
\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",1\right\rangle 
\end{matrix}\right\} \cap\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",1\right\rangle \\
\left\langle "Z",1\right\rangle 
\end{matrix}\right\} =\left\{ \begin{matrix}\left\langle "X",1\right\rangle \\
\left\langle "Y",1\right\rangle \\
\left\langle "Z",1\right\rangle 
\end{matrix}\right\} 
$