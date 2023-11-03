# De_novo_virus

Ce projet a pour but de réaliser une séquence consensus du génome d'un virus à partir de données séquencées obtenues par capture. Vous trouverez dans les différents dossiers : 
- DATA_CLEANING
- Trinity
- Blastn
- CAP3
- READ_MAPPING

## DATA CLEANING
Le DATA_CLEANING peut être réalisé à partir de l'outil GeckO : https://github.com/GE2POP/GeCKO/

```
../../../runGeCKO.sh --workflow DataCleaning --workflow-path ../../../../GeCKO --config-file CONFIG/config_DataCleaning.yml --cluster-config CONFIG/cluster_config_DataCleaning.yml --jobs 20 --job-scheduler SLURM
```

Pour plus d'informations, voir le Readme de l'outil GeckO : https://github.com/GE2POP/GeCKO/tree/main/DATA_CLEANING

## Trinity 
Pour créer la séquence consensus, on concatène tous les génotypes ensemble c'est-à-dire on concatène tous les fichiers fastq R1 et R2 respectivement dans un seul fichier après le data cleaning. 

```
cat fichier1.fastq.gz fichier2.fastq.gz > fichier_concatene.fastq.gz
```

Puis, on exécute le trinity sur les fichiers précédents
```
sbatch --partition=agap_normal --job-name=Tri --wrap="module load trinityrnaseq/2.13.2; Trinity --seqType fq --left ..._R1.fastq.gz --right ..._R2.fastq.gz --output nom_virus_trinity --CPU 8 --max_memory 50G"

```
On obtient ensuite un fichier FASTA contenant un ou plusieurs contigs. 
Ensuite, pour pouvoir filtrer les contigs par la suite, on enlève les espaces du fichier FASTA de sortie et on les remplace par des tirets. 

```
sed 's/ /-/g' nom_virus_trinity.Trinity.fasta > nom_virus_trinity.Trinity_avec_tirets.fasta
```
Avant de pouvoir blaster, il faut réaliser cette action: 

```
makeblastdb -in nom_virus_trinity.Trinity_avec_tirets.fasta -dbtype nucl
```

## Blast

```
sbatch --partition=agap_normal --job-name=Blastn --wrap="module load ncbi-blast/2.12.0+; blastn -task blastn -query /lustre/.../01_raw_data/REFERENCES/...fasta -db /lustre/.../02_results/.../.../Trinity/nom_virus_trinity.Trinity_avec_tirets.fasta -out /lustre/.../02_results/.../.../Blastn/blast_assemblage_trinity_sur_nom_virus.txt -outfmt '6 qseqid sseqid qstart qend qlen sstart send slen length pident evalue nident'
```

## Filtration des contigs
On filtre ensuite les contigs de sorte à n'avoir que des contigs avec des pvalues inférieures à 10^-11 et des longueurs de contigs supérieures à 100 bases. 
```
awk -F '\t' '$11 < 1e-11 && $9 > 100' blast_assemblage_trinity_sur_nom_virus.txt > blast_assemblage_trinity_sur_nom_virus_filtered.txt
```

Créer un fichier texte pour pouvoir coller le nom des contigs à conserver : ceux qui ont été filtrés par Blast. Pour cela, on copie colle la deuxième colonne de la sortie du blast qui correspond aux contigs filtrés : 

```
awk -F'\t' '{print $2}' blast_assemblage_trinity_sur_nom_virus_filtered.txt 
```
Coller la sortie dans un fichier sous format texte s'appelant : contigs_nom_virus_RNA1.txt & contigs_nom_virus_RNA2.txt 
Si votre virus possède deux ARN il faut dissocier les contigs pour chaque ARN en regardant la taille des séquences. Dans notre cas, les contigs se mappant sur une référence de 7000 bp étaient associés à ARN1 et ceux se mappant sur une référence de 4000 bp à l'ARN2. 

Ensuite, on vient filtrer le trinity obtenu précedemment : 
```
module load samtools/1.14-bin
samtools faidx nom_virus_trinity.Trinity_avec_tirets.fastac -r contigs_nom_virus_RNA1.txt > nom_virus_RNA1_filtered.fasta
samtools faidx nom_virus_trinity.Trinity_avec_tirets.fastac -r contigs_nom_virus_RNA2.txt > nom_virus_RNA2_filtered.fasta
```
Notre fichier FASTA contient maintenant que des contigs ayant une longueur supérieure à 100 paires de bases et une pvalue inférieure à 10^-11. 

Ce fichier va permettre de créer le consensus 

## CAP3

Pour créer le consensus, c'est-à-dire aligner les contigs du fichier FASTA en un seul fichier FASTA qui servira de référence on utilise l'outil CAP3. 
```
module load cap3/20151002
cap3 nom_virus_RNA1_filtered.fasta
cap3 nom_virus_RNA2_filtered.fasta
```
Si notre virus est connu pour avoir des insertions délétions il est possible de jouer sur la spécificité de l'assembleur en ajoutant des options par exemple : 

```
cap3 nom_virus_RNA1_filtered.fasta -e 5
```

On obtient un fichier .contigs qui est en réalité un fichier FASTA. Ce fichier est notre séquence consensus c'est-à-dire notre (nouvelle) référence. 

## READ_MAPPING
Une fois que le consensus est crée, on vient mapper les reads issus du data cleaning sur la séquence consensus. Pour le read mapping voir également GeckO : https://github.com/GE2POP/GeCKO/tree/main/READ_MAPPING

```
sbatch --partition=agap_normal --mem=10G --job-name=RM --wrap="/home/.../scratch/03_scripts/GeCKO/runGeCKO.sh --workflow ReadMapping --workflow-path /home/.../scratch/03_scripts/GeCKO/ --config-file CONFIG/config_ReadMapping_nom_virus.yml --jobs 20 --job-scheduler SLURM"
```

Dans le fichier : config_ReadMapping_nom_virus.yml, il faut mettre le fichier FASTA correspondant au consensus comme référence. 

Si l'on souhaite ensuite visualiser les alignements sur Tablet, il faut indexer les fichiers bams. 
```
for bam in $(ls *.bam); do samtools index ${bam}; done
```



