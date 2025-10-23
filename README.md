# longread_rcc
oxford ont long read analysis of ccRCC

```
# Path to your input file
input_file="SRR_Acc_List.txt"

# Declare an array
accession_array=()

# Read file line by line into the array
while IFS= read -r line || [[ -n "$line" ]]; do
    accession_array+=("$line")
done < "$input_file"

# Example: Print all elements
for acc in "${accession_array[@]}"; do
    echo "$acc"
done

# Loop through each accession
for acc in "${accession_array[@]}"; do
    echo "Prefetching $acc..."
    prefetch "$acc"  # download .sra file
    echo "Converting $acc to FASTQ..."
    fasterq-dump $acc/$acc.sra --split-files --outdir $acc  # convert to paired fastq
    echo "Finished $acc"
done


docker run -it --rm \
  --workdir $HOME \
  -v /mnt/g/reference:$HOME/reference \
  -v /mnt/h/scratch/longread_rcc:$HOME/longread_rcc \
  garywang7/isoseq:1.0.4 \
  /bin/bash


#!/bin/bash
#!/bin/bash

# Target directory
target_dir="longread_rcc"

# Output config file path
output_config="${target_dir}/isoquant_config.json"

# Find all .fastq files (not .fastq.gz) in longread_rcc
fastq_files=($(find "$target_dir" -type f -name "*.fastq" | sort))

# Start JSON array
{
echo "["
echo "  {"
echo "    \"data format\": \"fastq\""
echo "  },"
echo "  {"
echo "    \"name\": \"RAGE24_LongRead\","

# Add long read file paths
echo "    \"long read files\": ["
for i in "${!fastq_files[@]}"; do
  path="$(realpath "${fastq_files[$i]}")"
  if [[ $i -lt $((${#fastq_files[@]} - 1)) ]]; then
    echo "      \"$path\","
  else
    echo "      \"$path\""
  fi
done
echo "    ],"

# Add labels based on SRA accession prefix
echo "    \"labels\": ["
for i in "${!fastq_files[@]}"; do
  file=$(basename "${fastq_files[$i]}")
  label=$(echo "$file" | grep -oE 'SRR[0-9]+')
  if [[ -z "$label" ]]; then
    label="unknown_label_$i"
  fi
  if [[ $i -lt $((${#fastq_files[@]} - 1)) ]]; then
    echo "      \"$label\","
  else
    echo "      \"$label\""
  fi
done
echo "    ]"
echo "  }"
echo "]"
} > "$output_config"


ref_genome=reference/Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz
ref_gtf=reference/gencode.v45.annotation.gtf.gz
output_dir=longread_rcc/isoquant
num_threads=8
output_config=longread_rcc/isoquant_config.json

isoquant.py \
    -d ont \
    --yaml $output_config \
    --reference $ref_genome \
    --genedb $ref_gtf \
    --complete_genedb \
    --output $output_dir \
    --report_novel_unspliced false \
    --transcript_quantification with_ambiguous \
    --count_exons \
    --check_canonical \
    --sqanti_output \
    --threads $num_threads
```
