### **Assembly**

 1. Trim reads with [Trimmomatic v0.39](http://www.usadellab.org/cms/?page=trimmomatic) with the following parameters

`LEADING:3` TRAILING:15 SLIDINGWINDOW:4:10 MINLEN:36`

2. Assembly with [SPAdes v3.11.1](http://cab.spbu.ru/files/release3.11.1/) using the -meta flag.

__________________________________________________________________________

### **Gene Prediction with [Funannotate](https://funannotate.readthedocs.io/en/latest/)**

1. Sort by contigs by size and rename headers 

funannotate sort -i after_esom_assembly.fasta -o sorted_assembly.fasta -b profar

2. Mask repeats 

funannotate mask -i sorted_assembly.fasta -o masked_sorted_assembly.fas -s fungi

3.  Run gene prediction pipeline.

funannotate predict -i masked_sorted_assembly.fas --name IDENTIFIER_ --species "Species name" --protein_evidence /path/to/protein/evidence -o species_funannotate_out --cpus $SLURM_NTASKS 

### Phylogenomics Workflow
_**Note: This workflow uses nucleotide data (CDS files) as input and not amino acid data**_

1. Concatenate your original cds files and convert to a one line fasta 

`cat *.fasta | awk '/^>/ {printf("\n%s\n",$0);next; } { printf("%s",$0);}  END {printf("\n");}' > allcds_dna.fa`

2. Make a new directory and move your concatenated file

`mkdir cds_genes && mv allcds_dna.fa cds_genes`

3. Translate sequences to aa with EMBOSS

```
for f in *.fasta
do
~/EMBOSS-6.6.0/emboss/transeq -trim -sequence ${f} -out ${f%.fasta}.preaa.fasta
done

```
4. EMBOSS **stubbornly** adds as "_1" to each fasta header. I haven't found a way to turn this off. It will cause issues downstream if it's not removed.

```
for f in *.aln;
do
sed -i 's/T1_1/T1/g' $f; 
done
```

5. Run protein ortho

Run [ProteinOrtho](http://www.bioinf.uni-leipzig.de/Software/proteinortho/)

`proteinortho -clean -project=project_name -cpus=$SLURM_NTASKS *.aa.fasta`


6. Get single copy, shared orthologous genes. This examples has 108 input genomes.

`grep $'^108\t108' > single_copy_108.tsv`

Alternative: Check for a blank line up top. Use this command to remove if necessary with the sed command before grep.

`sed -n '1p' myproject.proteinortho.tsv | grep $'^108\t108' > single_copy_108.tsv`

7. Grab proteins from files with [src_proteinortho_grab_proteins.pl](https://github.com/tinamelie/Geoglossomycetes-genomics-workflow/blob/master/src_proteinortho_grab_proteins.pl). It's a good idea to take the header from the proteinortho output and include it at the top of your new tsv file. This can speed up the process.

`perl ./src_proteinortho_grab_proteins.pl -exact -tofiles new_all_108.tsv input_protein_file_1.fas input_protein_file_2.fas input_protein_file_3.fas`

8. Move the new single copy files to cds_genes then cd to that directory

`mv singlecopy108.tsv.OrthoGroup* cds_genes && cd $_`

9. Align the single copy files with MAFFT

```
for f in *fasta
do
mafft --thread 12 --amino --maxiterate 1000 ${f} > ${f%.fasta}.aln
done

```

10. Using your allcds_dna.fa, back translate to aa using RevTrans. This uses the match name to back translate, which is important so you do not lose the original translation. RevTrans is easy with a conda installation, but there's also a webserver.

```
for f in *aln
do 
revtrans.py -match name allcds_dna.fa ${f} > ${f%.aln}.revtrans.fasta
done
```

11. Run [trimAl](http://trimal.cgenomics.org/) with gappyout parameter

```
for f in *.revtrans.fasta
do 
trimal -in ${f} -out ${f%.revtrans}.trim
done
```

Note: If you encounter any problems from here, it’s because of the uneven sequence count making it “not an alignment”. All sequences must have the same number of bases to be considered an alignment. You can check a single file using this command:

`bioawk -c fastx '{ print $name, length($seq) }' inputfilename`

________________________________________________________________

### **Creating an ML tree**

1. Concatenate your fasta files. Be sure to check your headers here before moving forward

`catfasta2phyml.pl -f *trim > sequence.fasta`

2. Run [RAxML-NG](https://github.com/amkozlov/raxml-ng)

`raxml-ng --all sequence.fasta --model GTR+G --bs-trees 300`

________________________________________________________________

### **Creating ML gene trees with ASTRAL**

_This process uses the files in trimmed/ to prepare individual gene trees to then estimate a species tree with ASTRAL_

1. Run concatenated alignments of genes through RAxML to prepare individual gene trees

```
for f in *.trim
do
echo raxml-ng --all --msa ${f} --model GTR+G --bs-trees 100
done
```

2. Concatenate the resulting gene trees and clean up our folder a bit

```
mkdir genetrees && cd $_
mv ../*.support .
cat *.support > Astral_in.tre
```

3. Optional: Filter gene trees with low support branches using Newick utilities. Example filters anything below 30%. See ASTRAL paper for more information.

`nw_ed Astral_in.tre 'i & b<=30' o > Astral_in30.tre`

4. Run [ASTRAL](https://github.com/smirarab/ASTRAL). Output can be visualized as a species coalescent tree.

`java -jar astral.5.7.8.jar -i Astral_in30.tre -o Astral_out30.tre`

________________________________________________________________

### **Assessing Gene Conflict Using PhyParts**

1. Astral chooses an arbitrary taxon to root your tree to. It’s recommended that you manually reroot to your outgroup instead.

Example outgroup file:

`Saccharomyces cerevisiae`

Reroot your tree: 

`reroot_trees.py Astral_30out.tre outgroup.list > rerooted_Astral30.tre`

2. Run the PhyParts

```
java -jar phyparts-0.0.1-SNAPSHOT-jar-with-dependencies.jar -s 0.5 -a 1 -v -d ../support -m rerooted_Astral30.tre -o out

### -a = the type of analysis
### -v = verbose
### -m = target "species tree" from ASTRAL
### -d = directory with gene trees
### -s = bootstrap cutoff 50%
```

3. Visualize the results with phypartspiecharts. The number is the orthologous clusters/gene trees.

`phypartspiecharts.py reroot_astral.tre out 84`
