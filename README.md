# Guide des bonnes pratiques sous Unity

## Principe SOLID
[SOLID](https://fr.wikipedia.org/wiki/SOLID_%28informatique%29) est un acronyme qui représente cing principes ayant pour but de produire 
des architectures logicielles plus compréhensibles, flexibles et maintenables.
- Responsabilité unique (Single responsibility principle)
- Ouvert/fermé (Open/closed principle)
- Substitution de Liskov (Liskov substitution principle)
- Ségrégation des interfaces (Interface segregation principle)
- Inversion des dépendances (Dependency inversion principle)

### Application de principe d'inversion de dépendance
Imaginez le script suivant qui permet de gérer le personnage du jouer:
````cs
public class PlayerController : MonoBehaviour
{
    // ...
    
    void Update()
    {
        Movement();
    }

    private void Movement()
    {
        // On récupère l'entrée utilisateur
        // Cela va nous retourner -1 si on appuie sur la flèche de gauche et 1 sur la flèche de droite
        float movementInput = Input.GetAxis("Horizontal");
        
        // On utilise l'input pour déplacer notre joueur
    }
}
````
**Problème actuel:**

Il ya un fort couplage entre le script de joueur et les entrées utilisateurs. 
Notre code est ainsi:
- Très peu modulable, par exemple on pourrait vouloir des entrées clavier pour le joueur 1 et utiliser une manette pour le second. 
Il nous faudrait donc recoder un script de joueur à chaque fois.
- Très difficilement testable, le fort couplage rend quasi impossible les tests unitaires.
- Difficilement débugguable, car plus on a de dépendances et de code plus c'est dur de s'y retrouver. 

> **Inversion de dépendance (Wikipédia):**
>
> En suivant ce principe, la relation de dépendance conventionnelle que les modules de haut niveau ont, par rapport aux modules de bas niveau, est inversée dans le but de rendre les premiers indépendants des seconds.

Pour résumer, il faut utiliser les interfaces le plus possible.
Malheuresement, les interfaces ne sont pas supportés comme attribut Sérializable.
Afin de contourner le problème, nous pouvons tout simplement utiliser des MonoBehaviour ou des ScriptableObject avec des méthodes abstraites.

Nous allons donc appliquer ce principe aux entrées utilisateurs afin de reduire le couplage.
Cela nous permettrai notamment de remplacer les entrées utilisateurs par une IA s'il y a besoin.
Nous avons maintenant, tout ce qui concerne les inputs ici:
````cs
// Ici on créé une classe abstraite contenant la méthode abstraite servant à donner l'input horizontal
public abstract class IMovementInput : MonoBehaviour
{
    public abstract float GetHorizontalInput();
}

// Ici on définit l'implémentation concrète
// C'est ce script que l'on ajoutera sur l'objet dans la scène
public class KeyboardMovementInput : IMovementInput 
{
    public override float GetHorizontalInput() {
        return Input.GetAxis("Horizontal");
    }
}
````
Le script qui contrôle le joueur:
````cs
public class PlayerController : MonoBehaviour
{
    [SerializedField] private IMovementInput m_Input;
    // ...
    
    void Update()
    {
        Movement();
    }

    private void Movement()
    {
        // On récupère l'entrée utilisateur
        // Cela va nous retourner -1 si on appuie sur la flèche de gauche et 1 sur la flèche de droite
        float movementInput = m_Input.GetHorizontalInput();
        
        // On utilise l'input pour déplacer notre joueur
    }
}
````

### Application du principe de Responsabilité Unique
Imaginons que nous avons rajouté une mécanique de saut à notre personnage:
````cs
public class PlayerController : MonoBehaviour
{
    [SerializedField] private IMovementInput m_Input;
    // ...
    
    void Update()
    {
        Movement();
        Jump();
    }

    private void Movement() { ... }

    private void Jump() 
    {
        if(m_Input.ShouldJump) {
            // ...
        }
    }
}
````

> **Responsabilité unique**
> [...] every class in a computer program should have responsibility over a single part of that program's functionality, which it should encapsulate

En résumé une classe doit avoir une seule responsabilité, ce qui apporte plusieurs avantages:
- Les classes sont plus faciles à comprendre
- Donc plus facilement débugguable
- Elle n'a qu'une raison de ne pas fonctionner

On voit que l'on pourrait appliquer ce principe sur notre joueur: 
la mécanique de saut n'a absolument pas besoin de connaître le fonctionnement de la mécanique de mouvements latéraux.

On sépare alors les deux mécaniques en deux MonoBehaviour que l'on ajoutera sur le joueur:
````cs
public class PlayerMovement : MonoBehaviour
{
    [SerializedField] private IMovementInput m_Input;
    // ...
    
    void Update()
    {
        Movement();
    }

    private void Movement() { ... }
}

public class PlayerJump : MonoBehaviour
{
    [SerializedField] private IMovementInput m_Input;
    // ...
    
    void Update()
    {
        Jump();
    }

    private void Jump() { ... }
}
````

<details>
 <summary> Remarque </summary>
 Il faut aussi appliquer ce principe à notre classe qui gère les entrées utilisateur, 
 cela se fait dans la prochaine partie.
</details>


### Application du principe de Ségrégation d'interface
**Problème actuel dans les entrées utilisateurs:**
Imaginons que l'on souhaite implémenter les entrées du joueur de cette façon:
- Quand on appuie sur un bouton de l'UI on saute
- Quand on fait un swipe on va dans la direction souhaitée

Si l'on garde notre interface actuel, on va avoir une implémentation très complexe, 
or on a vu au dessus que c'était une très mauvaise idée.

> **Ségrégation d'interface (Wikipédia):**
>
> [...] aucun client ne devrait dépendre de méthodes qu'il n'utilise pas

Pour résumer, il faut que les interfaces ait une méthode ou une raison d'exister. Ce qui fait qu'une seule raison de casser.
Cela nous donnes les "interfaces" suivantes:
````cs
// Pour les entrées utilisateur liées au mouvement
public abstract class IMovementInput : MonoBehaviour
{
    public abstract float GetHorizontalInput();
}

// Pour les entrées utilisateur liées au saut
public abstract class IMovementInput : MonoBehaviour
{
    public abstract bool ShouldJump();
}
````

Ceci approte un autre intérêt: l'extensibilité. En séparant au maximum les fonctionnalités, 
vous pouvez très rapidement tester une nouvelle combinaison sans avoir à en recoder une entière.

### Application du principe Ouvert-Fermé
Imaginons que dans notre jeu, nous ayons un système de plaque de pression qui va déclencher un événement.
Tout d'abord nous faisons disparaître un objet:
````cs
public class Trigger : MonoBehaviour
{
    [SerializeField] private GameObject m_ObjectToInteractWith;
    
    private void OnTriggerEnter() {
        // On fait "disparaître" l'objet
        m_ObjectToInteractWith.SetActive(false);
    }
} 
````

Maintenant, on souhaite pouvoir déplacer un objet à certains endroits. 
Il nous faut donc les deux fonctionnalités:
````cs
// On utilise une enum pour spécifier le type d'intéraction que l'on veut
enum InteractionType {
    Disappear,
    Move,
}

public class Trigger : MonoBehaviour
{
    [SerializeField] private InteractionType m_Interaction;
    [SerializeField] private GameObject m_ObjectToInteractWith;

    // Utile que si interaction vaut Move
    [SerializeField] private Vector3 m_Movement;
    
    private void OnTriggerEnter() {
        switch(m_Interaction) {
            case Disappear:
                // On fait "disparaître" l'objet
                m_ObjectToInteractWith.SetActive(false);
                break;
            case Move:
                m_ObjectToInteractWith.transform.position += m_Movement;
                break;
        }
    }
} 
````

Comme vous pouvez le remarquez, il y a plusieurs problèmes dans le système actuel:
- Si l'on souhaite ajouter une nouvelle intéraction on doit modifier l'ancien code, 
il n'est ni fermé ni extensible (Principe ouvert-fermé de SOLID).
- Actuellement, si on veut juste faire disparaître un objet, 
le script connait quand même le vecteur de déplacement.
Cela risque de provoquer une certaine incompréhension 
à la personne qui pourrait l'utiliser dans l'éditeur par exemple.

Nous devrions appliquer deux principes ici:
- L'inversion de dépendance déjà évoquée
- Le principe ouvert-fermé

> **Ouvert-Fermé (Wikipédia):**
>
> [...] une classe doit être à la fois ouverte (à l'extension) et fermée (à la modification)

Pour résumer, il faut éviter les Switch Case ou les GetComponent en série dans Unity. 
Vous pouvez remplacer ces mécanismes par le polymorphisme, en utilisant les interfacs ou les MB abstraits.

En appliquant ce principe nous obtenons:
- D'un côté tout ce qui concerne la façon d'intéragir(mouvement, disaparition, etc.)
````cs
public abstract class ITriggerable : MonoBehaviour
{
    public abstract void Trigger();
}

public class DisappearTriggerable : MonoBehaviour
{
    public override void Trigger() {
        gameObject.SetActive(false);
    }
}

public class MoveTriggerable : MonoBehaviour
{
    [SerializeField] private Vector3 m_Movement;

    public override void Trigger() {
        transform.position += m_Movement;
    }
}

````
- De l'autre ce qui les déclenche
````cs
public class Trigger : MonoBehaviour
{
    [SerializeField] private ITriggerable m_Triggerable;
    
    private void OnTriggerEnter() {
        m_Triggerable.Trigger();
    }
} 
````

### Application du principe de substitution de Liskov



## Utilisation des ScriptableObject
Un ScriptableObejct(SO) est un peu comme un MonoBehaviour que vous pouvez enregistrer dans la hiérarchie de projet.
Vous codez une classe qui va représenter les données et les méthodes, quand un MB.
Ensuite, vous pourrez créer des instances très facilement, qui seront modifables dans l'éditeur d'Unity.
Cela a donc plusieurs avantages:
- Vous pouvez réduire le couplage dans votre code. 
En effet, plus besoin d'avoir un lien entre deux systèmes pour accéder à un seul paramètre/état.
Pour cela vous pouvez stocker des consantes, ou bien encore l'état d'un élément qui va changer dans le temps.
- Tout ce qui est modifié dans le SO quand on est en Play est enregistré.
- Les SO ne sont pas dépendant d'une scène à une autre
- Les SO ne possèdent pas les méthodes comme Update des MB, ils sont donc plus performants.

Vous pouvez regarder ce [tutoriel](https://www.youtube.com/watch?v=PVOVIxNxxeQ) si vous le souhaitez. 
Il résume bien ce que j'ai évoqué au-dessus et comment les créer/utiliser.

### Accéder à une valeur sans avoir de fortes dépendances
Imaginons que de nombreux scripts aient besoin d'accéder au Joueur (son transform, son gameobject, ses scripts etc.).
La solution la plus évidente est de créer un lien vers le joueur dans les scripts qui en ont besoin.
Mais cela lève un problème, il faut s'assurer que les bons objets ont été glissés au bon endroit, s'assurer que l'on en a oublié aucun, etc.
Pire, et si vous n'utilisez plus le même joueur. Vous devez donc retrouver toutes les références.

Il y a une solution très rapide à ça: l'utilisation d'un SO.
Vous aurez un SO qui ne contiendra qu'une variable qui représentera votre joueur. Au moment de sa création , le joueur s'affectera dedans.
Tous les scripts peuvent ensuite y accéder très facilement.

### Externaliser les données d'une classe
Les So sont très pratiques pour externaliser des données. Par exemple, vous avez un ensemble de personnages 
dans votre jeu. Ils ont tous un script qui gère les mouvements qui contient des variables pour la vitesse, etc.

Problème: Vous désirez changer la vitesse. Vous devez donc la changer sur tous vos personnages.

En utilisant, un SO vous réglez le problème. Toute votre configuration sera à un seul endroit.

### Réduire le couplage
Vous souhaitez ajouter de la vie à votre personnage ainsi qu'une barre de vie. 
Afin de pouvoir changer l'interface en fonction de la valeur vous devriez avoir un lien entre le joueur et l'interface.
Ceci serait très mauvais en terme d'extensibilité. De plus, si votre joueur n'est pas dans la même scène que l'UI
vous ne pourrez pas.

La solution est d'utiliser les SO (encore une fois...). 
Vous pouvez tout simplement stocker la vie dans un SO qui sera alors accessible par l'UI, mais aussi par n'importe quel autre composant.

<details>
 <summary> Problème de cette solution </summary>
Actuellement, pour savoir s'il y a un changement dans la valeur vous pouvez le vérifier dans Update.
Cette technique ne serait pas performante.
A la place vous pouvez utiliser les events qui s'implémenteront à l'aide des Delegate.
Je laisse les curieux aller voir comment les utiliser (et me demander s'il y a des soucis de compréhensions).
</details>

### Programmer des comportements
Comme évoqué dans l'introduction, les SO n'ont pas les méthodes comme Update et compagnie et sont donc plus performantes.
Vous pouvez notamment les utiliser pour programmer des comportements de façon très SOLID.
Par exemple, vous implémentez une arme dans votre jeu et vous aimeriez pouvoir soit tirer une balle rapide soit en tirer plusieurs lentes.
Au lieu de programmer ces comportements dans la classe de tir vous pouvez l'externaliser.
Vous aurez alors un ScriptableObject abstrait qui définit une méthode abstraite (SpawnBullet) puis
vous créerez des ScriptableObject concrets qui définiront les comportements. Vous avez ainsi un système très modulable 
et en plus très facilement utilisable par un no-coder.

## Utilisation de l'asset Atom
Nous avons évoqué ci-dessus l'utilisation des SO pour partager des variables entre plusieurs objets.
Nous avons aussi parlé de la notion d'Event. Nous pourrions recoder tous ces systèmes nous même mais quelqu'un 
a déjà développé un asset d'excellent qualité [Atom](https://github.com/unity-atoms/unity-atoms).
Je vous invite à aller voir sur le github comment l'installer et l'utiliser. Je ne peux que vous conseiller
de k'utiliser au maximum et suivre les principes qu'il évoque.

## Autres bonnes pratiques
- Attention à l'ordre des opérateurs
````cs
// Ici la multiplication se fait d'abord entre le vector et le premier flottant 
// puis entre le résultat et un flottant
// Ce qui fait 6 multiplications
return vector3 * float1 * float2;

// Ici on fait d'abord la multiplication entre les flottants
// Ce qui fait 4 multiplications
return vector3 * (float1 * float2);  
````

- Eviter les méthodes prenant une string en paramètre quand il s'agit des animator ou des material
````cs
 // Vous désirez changer la couleur d'un material et la propriété s'appelle _Color
material.SetColor("_Color", Color.red);

// Il est plus judicieux de faire
// Dans les variables de classe
private int m_ColorShaderId = Shader.PropertyToID("_Color")

// Puis l'utiliser
material.SetColor(m_ColorShaderId, Color.red)
````
<details>
Unity n'utilise pas directement les string pour changer les properties du material. 
En réalité, à chaque appel de la méthode avec une string, il va recalculer le hash en appelant Shader.PropertyToID.
Le même principe s'applique à l'utilisation de l'Animator par script.
</details>

- Stocker la valeur de Camera.main
<details>
Camera.main n'est pas une variable c'est une property. 
A chaque fois que l'on y accède, Unity fait FindObjectWithTag("MainCamera") qui est trop lourd.
Il vaut donc mieux y accéder une seule fois dans le Awake et la stocker dans une variable membre.
</details>

- Eviter les méthodes Unity qui prennent une string en paramètre (FindObjectWithTag, FindObjectByName, SetTrigger, etc.)
, elles sont très gourmandes.

## Quelques principes de base intéressants
Voici quelques principes qui sont intéressants à garder en tête quand on programme pour unity 
([tirés de ce lien](https://www.esprit-unity.fr/nouvelle-technique-pout-structurer-son-projet-unity3d/)).

### Être modulaire :
- Tous les composants doivent être indépendants les uns des autres
- Un non développeur doit pouvoir assembler ces composants pour créer des comportements non prévus à la base *

### Être éditable par tout le monde :
- Possibilité de changer le comportement sans toucher au code *
- Possibilité de tout modifié en mode Play

### Être simple à déboguer :
- Composants faciles à isoler pour les tester
- Avoir tous les outils pour déboguer un composant
- Ne jamais corriger un bug qu’on ne comprend pas


## Astuces


## Liens

### Talks
- [Talk sur l'utilisation des Scriptable Object pour améliorer l'architecture](https://www.youtube.com/watch?v=6vmRwLYWNRo)
- [Talk sur l'utilisation massive des SO (a inspiré l'asset Atom)](https://www.youtube.com/watch?v=raQ3iHhE_Kk)
- [Talk sur l'application des principes SOLID dans Unity](https://www.youtube.com/watch?v=eIf3-aDTOOA)
- [Talk de Sébastien Benard (DeepNight) sur le gameplay de Dead Cells](https://www.youtube.com/watch?v=M2jf_ST4Ez4)

### Liens en tout genre
- [Brackeys](https://www.youtube.com/channel/UCYbK_tjZ2OrIZFBvU6CCMiA) propose un grand nombre de tutoriels d'ecellentes qualités
- [Febucci](https://www.febucci.com/unity-tips/) regroupe de nombreuses astuces pour Unity
- [Ronja](https://www.ronja-tutorials.com/) propose des tutoriels sur l'écriture de shader (sans shader graph)