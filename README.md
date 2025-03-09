# Compte Rendu TP INFO805 Elias CUZEAU

## Introduction
Ce document décrit les choix d'implémentation pour la génération de code assembleur à partir d'un arbre syntaxique abstrait (AST). L'objectif est de convertir une structure de programmation de haut niveau en instructions en assembleur x86 tout en respectant une structure lisible et optimisée.

## Usage :

```shell
gradle build
java -jar ./build/libs/TP.jar ex1.lda
```

Vous trouverez différents fichier tests *.lda* dans le répertoire :
-*ex1.lda* : Code sur prixHt et prix Ttc
-*ex2.lda* : Code du calcul de PGCD avec while
-*ex3.lda* : Code de démonstration du if else then (je n'ai malheureusement pas implémenté les appels récursifs)

L'éxecution affichera l'arbre ainsi que le code compilé :
```shell
$java -jar ./build/libs/TP.jar ex1.lda
Arbre final :
└── ;
    ├── LET
    │   ├── prixHt
    │   └── 200
    └── LET
        ├── prixTtc
        └── /
            ├── *
            │   ├── prixHt
            │   └── 119
            └── 100

DATA SEGMENT
        prixHt DD
        prixTtc DD
DATA ENDS
CODE SEGMENT
        mov eax, 200
        mov prixHt, eax
        mov eax, prixHt
        push eax
        mov eax, 119
        pop ebx
        mul eax, ebx
        push eax
        mov eax, 100
        pop ebx
        div ebx, eax
        mov eax, ebx
        mov prixTtc, eax
CODE ENDS
```
## 1. **Fonction `compile(Node n)`**
La fonction `compile`  appellée en première sur la racine de l'arbre, et va s'appellée récursivement sur toute les instructions. Elle va ensuite **traduire les structures de contrôle** (`IF`, `WHILE`, `LET`, `OUTPUT`) en assembleur. Les expressions seront elles evaluées par la fonction `evaluate`.

```java
public void compile(Node n){
    switch(n.value){
        case ";":
            compile(n.left);
            compile(n.right);
            break;
        // ...
    }
}
```

### **1.1 Assignation (`LET`)**
```java
case "LET":
    evaluate(n.right);
    codeSegmentInstructions.add("\tmov " + n.left.value + ", eax\n");
    if (!variables.contains(n.left.value)) {
        dataSegmentInstructions.add("\t" + n.left.value + " DD\n");
        variables.add(n.left.value);
    }
    break;
```
On évalue l'expression et on stocke la valeur dans la variable cible. Si la variable n'existe pas, elle est ajoutée au segment de données.

### **1.2 Boucles (`WHILE`)**
Les boucles utilisent des **labels dynamiques** pour éviter les conflits d'étiquettes.
```java
case "WHILE":
    int currentLabel = labelCounter++;
    codeSegmentInstructions.add("debut_while_" + currentLabel + ":\n");
    evaluate(n.left);
    codeSegmentInstructions.add("\tjz fin_while_" + currentLabel + "\n");
    compile(n.right);
    codeSegmentInstructions.add("\tjmp debut_while_" + currentLabel + "\n");
    codeSegmentInstructions.add("fin_while_" + currentLabel + ":\n");
    break;
```
### **1.3 Conditions (`IF-THEN-ELSE`)**
```java
case "IF":
    int ifLabel = labelCounter++;
    evaluate(n.left);
    codeSegmentInstructions.add("\tjz else_" + ifLabel + "\n");
    compile(n.right.left);
    codeSegmentInstructions.add("\tjmp fin_if_" + ifLabel + "\n");
    codeSegmentInstructions.add("else_" + ifLabel + ":\n");
    compile(n.right.right);
    codeSegmentInstructions.add("fin_if_" + ifLabel + ":\n");
    break;
```

### **1.4 Affichage (`OUTPUT`)**
```java
case "OUTPUT":
    evaluate(n.left);
    codeSegmentInstructions.add("\tout eax\n");
    break;
```

## 2. **Fonction `evaluate(Node n)`**
La fonction `evaluate` est responsable de la **traduction des expressions arithmétiques et logiques** en code assembleur. Elle suit les étapes suivantes :

### **2.1 Détection du type de nœud**
Si `n.value` est une constante numérique (`NUMERIC`), elle est chargée directement dans `eax` :
```java
case "NUMERIC":
    codeSegmentInstructions.add("\tmov eax, " + n.value + "\n");
    break;
```
Si c'est une variable (`ALPHANUMERIC`), on la charge aussi dans `eax`, ou on gère le cas particulier de l'entrée utilisateur :
```java
case "ALPHANUMERIC":
    switch(n.value){
        case "INPUT":
            codeSegmentInstructions.add("\tin eax\n");
            break;
        default:
            codeSegmentInstructions.add("\tmov eax, " + n.value + "\n");
            break;
    }
    break;
```

### **2.2 Évaluation des opérations arithmétiques**
Les opérations `+`, `-`, `*`, `/` sont effectuées en **empilant** la première opérande (`push eax`), évaluant la seconde, puis en appliquant l'opération correspondante.
Par exemple, pour l'addition :
```java
case "+":
    evaluate(n.left);
    codeSegmentInstructions.add("\tpush eax\n");
    evaluate(n.right);
    codeSegmentInstructions.add("\tpop ebx\n");
    codeSegmentInstructions.add("\tadd eax, ebx\n");
    break;
```
Même logique pour `-`, `*` et `/`.

### **2.3 Comparaisons (`<`, `>`, `<=`, `>=`)**
On utilise `sub` suivi de la bonne instruction de saut (`jl`, `jg`, `jle`, `jge`). Un **compteur de labels** est utilisé pour éviter les conflits dans plusieurs comparaisons.
```java
case "<":
case ">":
case "<=":
case ">=":
    int labelId = labelCounter++;
    evaluate(n.left);
    codeSegmentInstructions.add("\tpush eax\n");
    evaluate(n.right);
    codeSegmentInstructions.add("\tpop ebx\n");
    codeSegmentInstructions.add("\tsub ebx, eax\n");
    String jumpInstruction = "";
    switch (n.value) {
        case "<": jumpInstruction = "jl"; break;
        case ">": jumpInstruction = "jg"; break;
        case "<=": jumpInstruction = "jle"; break;
        case ">=": jumpInstruction = "jge"; break;
    }
    codeSegmentInstructions.add("\t" + jumpInstruction + " vrai_" + jumpInstruction + "_" + labelId + "\n");
    codeSegmentInstructions.add("\tmov eax, 0\n");
    codeSegmentInstructions.add("\tjmp fin_" + jumpInstruction + "_" + labelId + "\n");
    codeSegmentInstructions.add("vrai_" + jumpInstruction + "_" + labelId + ":\n");
    codeSegmentInstructions.add("\tmov eax, 1\n");
    codeSegmentInstructions.add("fin_" + jumpInstruction + "_" + labelId + ":\n");
    break;
```


