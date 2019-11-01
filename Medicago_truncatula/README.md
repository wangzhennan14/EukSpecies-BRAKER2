# Species: _Medicago truncatula strain:A17 (barrel medic)_  
Alex Lomsadze  
Georgia Institute of Technology  
2019  
## Project setup  
```
species="Medicago_truncatula"
base="/storage3/EukSpecies"
export PATH="$base/bin:$PATH"
export base="$base/$species"
cd $base
if [ "$(pwd)" != "$base" ]; then echo "error, folder not found: $base"; fi
umask 002
```
Create core folders  
```
cd $base
mkdir -p arx annot data
```
### Genome sequence  
Assembly description is at https://www.ncbi.nlm.nih.gov/assembly/GCF_000219495.3  
```
# download data
cd $base/arx
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/219/495/GCF_000219495.3_MedtrA17_4.0/GCF_000219495.3_MedtrA17_4.0_genomic.fna.gz
gunzip  GCF_*.fna.gz

# create ID table
grep '^>' GCF_*.fna > deflines
# move sequence IDs to chr1 style
cat deflines | grep -Ev '^>NW_|^>NC_003119'   | cut -f1,7 -d' ' | cut -b2- | sed 's/ / chr/' | tr -d ',' > list.tbl

# select and reformat sequence; all uppercase 
get_fasta_with_tag.pl --swap --in GCF_*_genomic.fna  --out tmp_genome.fasta  --list list.tbl --v
probuild --reformat_fasta --in tmp_genome.fasta --out genome.fasta --uppercase 1 --letters_per_line 60 --original

# check sequence and clean folder
probuild --stat --details --seq tmp_genome.fasta
probuild --stat --details --seq genome.fasta

# put in work folder
mv genome.fasta ../data/genome.fasta

# clean tmp files
rm tmp_genome.fasta
gzip  GCF_*_genomic.fna
```
### Masking: _de novo_ and _species specific_
Run _de novo_ masking of genome using RepeatModeler.  
Run this on AWS node configured for RM:  
    ec2-13-59-253-165.us-east-2.compute.amazonaws.com  
```
ssh  alexl@ec2-13-59-253-165.us-east-2.compute.amazonaws.com
# set the environment
umask 002
species="Medicago_truncatula"
cd /data
mkdir -p $species
cd $species
mkdir -p data RModeler RMasker
cd data
scp alexl@topaz.gatech.edu:/storage3/EukSpecies/$species/data/genome.fasta  .
  ## password
cd /data/$species/
cp ../bin/run_masking.sh .
nohup ./run_masking.sh >&  loginfo &
# wait and check
cd RMasker
scp  genome.fasta.masked  alexl@topaz.gatech.edu:/storage3/EukSpecies/$species/data
  ## password
exit
```
Get masking coordinates from soft-masked sequence 
```
cd $base/annot/
soft_fasta_to_3 < ../data/genome.fasta.masked | awk '{print $1 "\tsoft_masking\trepeat\t" $2+1 "\t" $3+1 "\t.\t.\t.\t." }' > mask.gff
```
### Annotation  
Download annotation from http://www.medicagogenome.org  
Select only protein coding genes from annotation and save it in GFF3 and GTF (stop codon included) formats.  
```
# download
cd $base/arx
wget https://de.cyverse.org/dl/d/495B7D67-19AB-4C42-ABA8-8A146C9DCDBB/Mt4.0v2_genes_20140818_1100.gff3
gunzip Mt4.0v2_genes_20140818_1100.gff3

# select 
gff_to_gff_subset.pl --in Mt4.0v2_genes_20140818_1100.gff3 --out annot.gff3 --list list.tbl --col 2

# reformat into "nice" gff3
echo "##gff-version 3" > tmp_annot.gff3
probuild --stat_fasta --seq ../data/genome.fasta | cut -f1,2 | tr -d '>' | grep chr | awk '{print "##sequence-region  " $1 "  1 " $2}' >> tmp_annot.gff3
cat annot.gff3 | grep -v gff-version  >> tmp_annot.gff3
mv tmp_annot.gff3 annot.gff3

# check
/home/tool/gt/bin/gt  gff3validator annot.gff3

# make nice
/home/tool/gt/bin/gt  gff3  -force  -tidy  -sort  -retainids  -checkids  -o tmp_annot.gff3  annot.gff3
mv tmp_annot.gff3  annot.gff3

# separate pseudo
select_pseudo_from_nice_gff3.pl annot.gff3 pseudo.gff3

# add features
enrich_gff.pl --in annot.gff3 --out ref.gff3 --cds --seq ../data/genome.fasta --v --warnings
gff3_to_gtf.pl ref.gff3 ref.gtf

# check
compare_intervals_exact.pl --f1 annot.gff3  --f2 ref.gff3
compare_intervals_exact.pl --f1 annot.gff3  --f2 ref.gtf
/home/braker/src/eval-2.2.8/validate_gtf.pl  ref.gtf

# move files to annot folder
mv ref.gff3     ../annot/annot.gff3
mv ref.gtf      ../annot/annot.gtf
mv pseudo.gff3  ../annot/

rm annot.gff3
gzip Mt4.0v2_genes_20140818_1100.gff3
```
