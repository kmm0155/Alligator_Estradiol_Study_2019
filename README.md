Title: Xenobiotic estradiol-17ß alters the microbial gut communities of hatchling American Alligators (Alligator mississippiensis)

Kaitlyn M. Murphy1, Madison M. Watkins1, John W. Finger1, Meghan D. Kelley1, Ruth M. Elsey2, Daniel A. Warner1, Mary T. Mendonça1

1Department of Biological Sciences, Auburn University, Auburn, AL 36849
2Louisiana Department of Wildlife and Fisheries, Grand Chenier, LA 

#This script (along with the attached text version) will give you the output for future analyses involved in this manuscript. For additional questions, feel free to email me at kmm0155@auburn.edu.

#!/bin/sh

cd Desktop

    source activate qiime2-2020.2
	 
#https://docs.qiime2.org/2018.6/tutorials/overview/#taxonomy-classification-and-taxonomic-analyses
#https://sites.google.com/site/knightslabwiki/qiime-workflow
#visualization at https://view.qiime2.org/

#To demultiplex your samples, use the following code. You will need to download
#Dr. Demuxy from GitHub at the following address: https://github.com/lefeverde/Mr_Demuxy
#You need to make sure that Mr_Demuxy is in your path. If you installed Miniconda, for #QIIME, you should already have pip. This is what you need to install Mr_Demux:
#pe_demuxer.py  -r1 R1.fastq -r2 R2.fastq -r2_bc reverse_bcs.txt -r1_bc forward_bcs.txt 

#I have paired end sequences and need to list the file as: #SampleData[PairedEndSequencesWithQuality]. R1 is forward and R2 is reverse.
#To see other types of 'type' files for QIIME tools import, see https://github.com/qiime2/q2cli/issues/124
#The demultiplexer did not work, I asked for it to be done. In the future, always have your samples demultiplexed.

#RUN THE CODE!
		
#create qiime aritfact
#https://docs.qiime2.org/2019.7/tutorials/importing/ 

qiime tools import \
	--type 'SampleData[PairedEndSequencesWithQuality]' \
	--input-path /Users/kaitlynmurphy/Desktop/Alligator_Data/Alligator_Manifest.csv \
	--output-path paired-end-seqs.qza \
	--input-format PairedEndFastqManifestPhred33
	 
#use this to peek at the data
qiime tools peek paired-end-seqs.qza

#denoises sequences, dereplicates them, and filters chimeras
#https://docs.qiime2.org/2018.6/plugins/available/dada2/denoise-pyro/

qiime dada2 denoise-paired \
	--i-demultiplexed-seqs paired-end-seqs.qza \
	--p-trim-left-f 20 \
	--p-trim-left-r 20 \
	--p-trunc-len-f 285 \
	--p-trunc-len-r 285 \
	--p-trunc-q 15 \
	--p-chimera-method consensus \
	--o-representative-sequences rep-seqs-denoise.qza \
	--o-table rep_seq_feature_table.qza \
	--o-denoising-stats denoising-stats.gza \
	--verbose
	
#summary stats of denoise and quality filtering

qiime feature-table summarize \
	--i-table rep_seq_feature_table.qza \
	--o-visualization rep_seq_feature_table-view.qzv \
	--m-sample-metadata-file /Users/kaitlynmurphy/Desktop/Alligator_Data/Alligator_Metadata.txt

#Feature table!	

qiime feature-table tabulate-seqs \
	--i-data rep-seqs-denoise.qza \
	--o-visualization rep-seqs-view.qzv

#Filter features from feature table
#features must be a minimum sum of 20 across all samples and must be present in at least 5 samples: https://docs.qiime2.org/2019.7/tutorials/filtering/
	
qiime feature-table filter-features \
	--i-table rep_seq_feature_table.qza \
	--p-min-frequency 10 \
	--p-min-samples 1 \
	--o-filtered-table rep_seq_feature_table2.qza
	
#Now filter sequences to match table 
#https://docs.qiime2.org/2018.8/plugins/available/feature-table/filter-seqs/

qiime feature-table filter-seqs \
	--i-data rep-seqs-denoise.qza \
	--i-table rep_seq_feature_table2.qza \
	--o-filtered-data rep-seqs-filtered.qza
	
	if [ ! -f rep-seqs-filtered.qza ];
	then
		echo "File not found!" && exit 0
	else
        continue
	fi
	
#summary stats of filtering

qiime feature-table summarize \
	--i-table rep_seq_feature_table2.qza \
	--o-visualization rep_seq_feature_table_filter-view.qzv \
	--m-sample-metadata-file /Users/kaitlynmurphy/Desktop/Alligator_Data/Alligator_Metadata.txt

#rarefaction curve

qiime diversity alpha-rarefaction \
	--i-table rep_seq_feature_table2.qza \
	--p-max-depth 197 \
	--m-metadata-file /Users/kaitlynmurphy/Desktop/Alligator_Data/Alligator_Metadata.txt \
	--o-visualization alpha-rarefaction.qzv

#Taxonomy Classification and taxonomic analyses
#https://www.arb-silva.de/fileadmin/silva_databases/qiime/Silva_132_release.zip
#https://docs.qiime2.org/2018.6/tutorials/feature-classifier/
#https://forum.qiime2.org/t/silva-132-classifiers/3698
#https://www.dropbox.com/s/5tv5uk95pk3ukwf/7_level_taxonomy.qza?dl=0

#Consensus must match 100% where majority is 90% to the cluster; 7 levels refers to taxonomic levels

qiime feature-classifier extract-reads \
	--i-sequences /Users/kaitlynmurphy/Desktop/QIIME_Analyses/99_otus.qza \
	--p-f-primer GTGCCAGCMGCCGCGGTAA \
	--p-r-primer GGACTACHVGGGTWTCTAAT \
	--o-reads gg-ref-seqs.qza

	if [ ! -f  gg-ref-seqs.qza ];
	then
		echo "File not found!" && exit 0
	else
        continue
	fi
	
#training sklearn

qiime feature-classifier fit-classifier-naive-bayes \
	--i-reference-reads gg-ref-seqs.qza \
	--i-reference-taxonomy /Users/kaitlynmurphy/Desktop/QIIME_Analyses/99_otu_taxonomy.qza \
	--o-classifier gg-99-classifier.qza
  
qiime feature-classifier classify-sklearn \
	--i-classifier gg-99-classifier.qza \
	--i-reads rep-seqs-filtered.qza \
	--o-classification classified_taxonomy_table.qza
	
qiime metadata tabulate \
	--m-input-file classified_taxonomy_table.qza \
	--o-visualization classified_taxonomy.qzv

#plug the below barplot into view.qiime2 to see what it looks like!

qiime taxa barplot \
	--i-table rep_seq_feature_table2.qza \
	--i-taxonomy classified_taxonomy_table.qza \
	--m-metadata-file /Users/kaitlynmurphy/Desktop/Alligator_Data/Alligator_Metadata.txt \
	--o-visualization taxa-barplots.qzv

qiime feature-table heatmap /
	--i-table rep_seq_feature_table2.qza /
	--m-metadata-file Egg_Metadata.txt /
	--o-visualization heatmap.qzv /
	--m-metadata-column subject	  
	
#qiime taxa collapse 

qiime taxa collapse \
	--i-table rep_seq_feature_table2.qza \
	--i-taxonomy classified_taxonomy_table.qza \
	--p-level 7 \
	--output-dir rep_seq_feature_table3_collapsed
	
	
	if [ ! -f rep_seq_feature_table3.qza ];
	then
		echo "File not found!" && exit 0
	else
       continue
	fi
	
#export data

qiime tools export \
	--input-path /Users/kaitlynmurphy/Desktop/rep_seq_feature_table3_collapsed/collapsed_table.qza \
	--output-path exported

qiime tools export \
	--input-path classified_taxonomy_table.qza \
	--output-path exported

if [ ! -f exported/feature-table.biom ];
	then
		echo "File not found!" && exit 0
	else
        continue
	fi

if [ ! -f exported/taxonomy.tsv ];
	then
		echo "File not found!" && exit 0
	else
        continue
	fi

#Go back into the MetaData file and add a '#' to the title line. This will allow biom to detect the header
	
biom add-metadata \
	-i feature-table.biom \
	-o Alligator_w_taxonomy.biom \
	--observation-metadata-fp Alligator_Metadata.txt \
	--sc-separated taxonomy

biom convert -i Alligator_w_taxonomy.biom -o Alligator_w_taxonomy.tsv --to-tsv

#Filter/separate treatment groups based on treatment in order to determine differential abundance

qiime feature-table filter-samples \
	--i-table /Users/kaitlynmurphy/Desktop/rep_seq_feature_table3_collapsed/collapsed_table.qza \
	--m-metadata-file Alligator_Metadata.txt \
	--p-where "subject='Control'" \
	--o-filtered-table Control-abundance-table.qza

qiime feature-table filter-samples \
	--i-table /Users/kaitlynmurphy/Desktop/rep_seq_feature_table3_collapsed/collapsed_table.qza \
	--m-metadata-file Alligator_Metadata.txt \
	--p-where "subject='Low'" \
	--o-filtered-table Low-abundance-table.qza

qiime feature-table filter-samples \
	--i-table /Users/kaitlynmurphy/Desktop/rep_seq_feature_table3_collapsed/collapsed_table.qza \
	--m-metadata-file Alligator_Metadata.txt \
	--p-where "subject='High'" \
	--o-filtered-table High-abundance-table.qza

##DIVERSITY##
#https://docs.qiime2.org/2018.11/plugins/available/diversity/

qiime diversity alpha \
  --i-table rep_seq_feature_table2.qza \
  --p-metric shannon \
  --o-alpha-diversity shannon_vector.qza

#Exports file in tsv format
qiime tools export \
  --input-path shannon_vector.qza \
  --output-path shannons
