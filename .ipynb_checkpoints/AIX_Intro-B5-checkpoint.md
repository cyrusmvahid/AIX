# A Gentle Introduction to AI Explainability - part 5: SHAP

## Context
In [part 4](./AIX_Intro-B4.md) we went through the theoretical foundation of Shapley values. In this part we focus on application of Shapley values in local explainability.

# Additive feature attribution methods
Additive feature attribution methods have a an explanation model that is a linear function of binary variables.
$$
\large{
g(z^{\prime}) = \phi_0 + \sum_{i=1}^M\phi_iz^\prime_i
}
$$
where $z^\prime \in \{0,1\}^M$, and $M$ is the number of simplified input features, and $\phi_i \in \mathbb{R}$.

Family of explanation models that matching definition of additive feature attribution models assign an attribution $\phi_i$ to each feature and then sum up the effects of all features to approximate the original model $f$ over a specific input. 
In the following sections we explain such models as well as their practical application. 

Additive feature attribution methods have *desirable* properties that uniquely determine additive feature attribution.

### Local accuracy
Local accuracy requires the explanation model $g$ to at least match the output of original model $f$ for the simplified input $x^\prime$.

### Missingness
features that are missing from the simplified input, which describe in binary terms where or or not a feature is present, must have no impact, or $x_i^\prime = 0 \Rightarrow \phi_i=0$.

### Consistency
If a model changes in a way that impact of a feature increases, the attribution should never decrease.

## SHAP for Explainability
So far we have seen that Shapley values provide a ***unique*** and ***fair*** way to distribute the payout amongst the players in a collaborative and competitive game. We have also seen references to feature attribution methods. Additionally we know from the context that SHAP is an additive feature attribution methods and thus has a linear local explanation based on simplified input.    

The authors of the SHAP paper[10] have provided an equation that is akin to Shapley values equation and prove that there is unique solution to that equation, which is composed of a conditional expectation function of the original model that is being explained. In this case the explanation model should satisfy the three properties of the additive feature attribution methods, which again are akin to three axioms of for Shapley theorm. The equation that needs to be answered is defined as:
$$
\large{
\phi_i(f,x) = \sum_{z^\prime \subseteq x^\prime} \frac{|z^\prime|!(M-|z^\prime|-1)!}{M!} \left[ f_x(z^\prime)-f_x(z^\prime \setminus i) \right]
}
$$
where $|z|$ is the number of non-zero entries in z, and $z \subseteq x$ represents all $z$ vectors where the non-zero entries are a subset of the non-zero entries in $x$. $f_x(z\prime) = E \left[ f(z) | z_S\right]$ where $S$ is a set of non-zero indexes in $z^\prime$

Comparing this equation to shapley values, we observe that $\phi = \{\phi_i:\ 0\ \leq i\ \leq M\}$. $f$, the original model is the payout function and $x$ denotes the grand coalition. finally $z$ is permutations of possible coalitions, which we saw in the simple example s we created those permutations. 

<figure>
    <img img align="center"  src='data/SHAP.png'>
    <figcaption>
        Figure 1: $[E(f(z)]$, corresponding to $\phi_0$ is the expectation of the model over input features. Then as we build all the possible alliances $x_1 = \rightarrow x_{1,2} \rightarrow x_{1,2,3}$ we get the conditional expectation  of $f(x)$ for the newly added feature. We should keep in mind that the solution is dependent on the order the population is generated in the cases where the features are dependent on one another or the model $f$ is non-linear.
    </figcaption>
</figure>

## Approximating SHAP values
Instead of heavy and complicated computation of SHAP values, we can approximate them with some accuracy. Authors of [10] has proposed two model agnostic methods of which we only focus on Kernel SHAP, which is a combination of LIME and Shapley values. In next postings, we describe DeepLift and DeepSHAP, which combined DeepLift an Shapley values and is model specific.

### Kernel SHAP
We remember creating a LIME explanator resulted in solving the optimization 

$$
\large{\xi(x) = \text{arg} \min\limits_{g \in G} \mathcal{L}(f,g,\Pi_x) + \Omega(g)}
$$.

We should also note that choosing loss function and $\Omega$, and weighting kernel, $\Pi$ to solve the LIME optimization equation are empirical and based on heuristic methods, thus the explanations can vary depending on how we choose those hyperparameters. The question is whether we could do better and have a consistent and locally accurate solution. The answer is yes. Since LIME is a Additive feature attribution method, Shapley values are the unique solution to the problem of finding an explanator where the desired properties, local accuracy, missingness, and consistency are satisfied; therefore the question of consistency and local accuracy for LIME comes down to find the Shapley values to to find the hyperparameters $\Omega$, $\mathcal{L}$, and $\Pi$ and avoid heuristic methods.
The authors propose and prove the following values for the huperparameters to be the unique solution that satisfies the desired properties:
$$
\large{
\Omega(g)=0
}
$$
As seen earlier $\Omega$ represented complexity such as depth of a tree or the number of non-zero weights. This is heuristic and arbitrary. By setting it to $0$, we become independent of such arbitrary choice of complexity.

$$
\large{
\Pi_{x^\prime}(z^\prime)=\frac{M-1}{\left(M choose |z^\prime| \right )|z^\prime|\left( |M|-|z^\prime|\right )}
}
$$
The apparent similarity to weighting of Shapley values is obvious.

$$
\large{
\mathcal{L}(f,g,\Pi_{x^\prime}) = \sum_{z^\prime \in z} \left[ f(h_x^{-1}(z^\prime))-g(z^\prime)\right ]^2\Pi_{x^\prime(z^\prime)}
}
$$
$\mathcal{L}$ corresponds to the weighted average of the conditional expectations in the SHAP method.


## Implementation Example

#### Loading an Image

```python
from PIL import Image
from torchvision import transforms
pil_img = Image.open("data/boyz2.jpeg")
pil_img.convert("RGB")

resize_transform  = transforms.Resize((224,224))
tensor_transform = transforms.ToTensor()


pil_img = resize_transform(pil_img)
plt.imshow(pil_img)
pil_img = tensor_transform(pil_img)#.numpy()
#normalize = transforms.Normalize(mean=[0.485, 0.456, 0.406],
#                                std=[0.229, 0.224, 0.225])  
#pil_img = normalize(pil_img)
pil_img = np.transpose(pil_img, (1,2,0))
pil_img = pil_img * 255
```

<figure>
    <img img align="center"  width="224" height="224" src='data/boyzplt.png''>
    <figcaption>
    </figcaption>
</figure>

#### Segmenting the image to 50 segments so we do not end up explaining every pixel.
```python
from skimage.segmentation import slic

segments_slic_pil = slic(pil_img, n_segments=50, compactness=30, sigma=3)
```

#### define a function that depends on a binary mask representing if an image region is hidden

```python
def mask_image(zs, segmentation, image, background=None):
    if background is None:
        background = image.mean((0,1))
    out = np.zeros((zs.shape[0], image.shape[0], image.shape[1], image.shape[2]))
    for i in range(zs.shape[0]):
        out[i,:,:,:] = image
        for j in range(zs.shape[1]):
            if zs[i,j] == 0:
                out[i][segmentation == j,:] = background
    return out
```

#### Creating a predictor that works on super-pixels as opposed to the original image.

```python
from keras.applications.vgg16 import VGG16
model = VGG15()
def f(z):
    print("inside f")
    return model.predict(preprocess_input(mask_image(z, segments_slic, pil_img, 255)))
```

#### Creating an explainer using shap library
```python
import shap
explainer = shap.KernelExplainer(f, np.zeros((1,50)))
```

#### Computing Shaply values using our explainer. 
We run VGG 100 times on the perturbed dataset.
```python
shap_values = explainer.shap_values(np.ones((1,50)), nsamples=1000) 
```

#### Visualization of the explanation
In explaining how the white dog has been identified as a west highland terrier, we can see the face (colour green) and the positive contribution of 0.1 has the highest impact. Notably we can also observe that the red zones have the most negative imact.

<figure>
    <img img align="center"  " src='data/boyzexplained.png'>
    <figcaption>
    </figcaption>
</figure>

You can find the full code [here](SHAPExample.md)


# References
1. https://arxiv.org/pdf/1602.04938v1.pdf
2. https://arxiv.org/pdf/2011.07876.pdf
3. https://www.oreilly.com/content/introduction-to-local-interpretable-model-agnostic-explanations-lime/
4. https://github.com/marcotcr/lime/tree/master/doc/notebooks
5. https://arxiv.org/pdf/1705.07874.pdf
6. https://vknight.org/Year_3_game_theory_course/Content/Chapter_16_Cooperative_games/
7. https://www.rand.org/content/dam/rand/pubs/papers/2021/P295.pdf
8. https://www.wifa.uni-leipzig.de/fileadmin/Fakultät_Wifa/Institut_für_Theoretische_Volkswirtschaftslehre/Professur_Mikroökonomik/Cooperative_game_theory/B1_gl.pdf
9. https://www.youtube.com/watch?v=9OFMRiAVH-w
10 https://arxiv.org/pdf/1705.07874.pdf

