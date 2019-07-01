# Species: _Neurospora_crassa_
Alex Lomsadze  
Georgia Institute of Technology  
2019  
## Project setup
Project is set in bash shell.  

Setup environment on GT cluster as:  
```
umask 002

base="/storage3/w/alexl/EukSpecies"
species="Neurospora_crassa"

export PATH="$base/bin:$PATH"
export base="$base/$species"
cd $base
if [ "$(pwd)" != "$base" ]; then echo "error, folder not found: $base"; fi
```
Create core folder structure
```
cd $base
mkdir -p arx annot data mask
```
Download genomic sequence and reformat it:  
 * simplified FASTA defline with a first word in defline as a unique sequence ID
 * select only nuclear DNA (exclude organelles)
 * set sequence in all uppercase
When possible use genomic sequence from NCBI.  
Match sequence ID in FASTA file with sequence ID in annotation file.  
Use ID from annotation.  
Keep IDs in the file "list.tbl".  
First column in the table is sequence ID and second column is annotation ID.  

Description of assembly is at https://www.ncbi.nlm.nih.gov/assembly/GCF_000182925.2  
```
cd $base/arx
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/182/925/GCF_000182925.2_NC12/GCF_000182925.2_NC12_genomic.fna.gz
gunzip  GCF_000182925*.fna.gz

grep '^>' GCF*.fna
grep '^>' GCF*.fna | grep -v '>NW_' | grep -v NC_026614 | cut -f1,7 -d' ' | tr -d ',' | tr -d '>' > list.tbl

# Attantion, modify manually sequence ID's in list.tbl to match the annoatation file

get_fasta_with_tag.pl --swap --in GCF_000182925.2_NC12_genomic.fna  --out genome.fasta  --list list.tbl --v
probuild --stat --details --seq genome.fasta
probuild --reformat_fasta --in genome.fasta --out ../data/genome.fasta --uppercase 1 --letters_per_line 60 --original
rm genome.fasta
probuild --stat --details --seq ../data/genome.fasta

gzip  GCF_000182925*.fna
```
Run _de novo_ masking of genome using RepeatModeler.  
Run this on AWS node configured for RM:  
    ec2-13-59-253-165.us-east-2.compute.amazonaws.com
```
ssh  alexl@ec2-13-59-253-165.us-east-2.compute.amazonaws.com
# set the environment
umask 002
species="Neurospora_crassa"

cd /data
mkdir -p $species
cd $species
mkdir -p data RModeler RMasker
cd data
scp alexl@topaz.gatech.edu:/storage3/w/alexl/EukSpecies/$species/data/genome.fasta  .
  ## password
cd ..
cp ../bin/run_masking.sh .
nohup ./run_masking.sh >&  loginfo &
# wait and check
cd RMasker
scp  genome.fasta.masked  alexl@topaz.gatech.edu:/storage3/w/alexl/EukSpecies/$species/data
  ## password
exit
```
Download annotation from JGI.  
JGI annotation is copy of Broad annotation.
https://genome.jgi.doe.gov/portal/pages/dynamicOrganismDownload.jsf?organism=Neucr2  
NCBI RefSeq may differ from Broad.  
Select only protein coding genes from annotation and save it in GFF3 and GTF (stop codon included) formats.  
```
cd $base/arx
# manual download
gunzip  Neucr2.filtered_proteins.BroadModels.gff3.gz

gff_to_gff_subset.pl  --in Neucr2.filtered_proteins.BroadModels.gff3  --out annot.gff3 --list list.tbl --col 2
#check
/home/tool/gt/bin/gt  gff3validator  annot.gff3
#reformat
/home/tool/gt/bin/gt  gff3  -force  -tidy  -sort  -retainids  -checkids  -o tmp_annot.gff3  annot.gff3
mv tmp_annot.gff3 annot.gff3

gff3_to_gtf.pl annot.gff3 annot.gtf
#check
/home/braker/src/eval-2.2.8/validate_gtf.pl -c  annot.gtf
compare_intervals_exact.pl --f1 annot.gff3  --f2 annot.gtf

mv annot.gff3 ../annot/
mv annot.gtf  ../annot/

gzip Neucr2.filtered_proteins.BroadModels.gff3
```

