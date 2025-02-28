# Morphogenomics
Steps followed in the Cephalobidae morphogenomics project. Phylogenetic analysis will be based on genome skimming data. Here an overview of the bioinformatics pipeline is given for the test dataset using Nanopore sequencing.

# Cephalobidae genome skims

---

## Initial test - fast basecalling

1. Basecalling with fast algorithm
```=shell!
./dorado-0.3.0-linux-x64/bin/dorado basecaller ./dorado-0.3.0-linux-x64/bin/dna_r10.4.1_e8.2_400bps_fast@v4.2.0 run1_13barcodes/20241128_1456_P2S-01326-B_PAQ51960_c746f836/pod5/ > Cephalob_skims_basecalling_fast.bam
```
3. Convert bam to fastq
```
samtools fastq Cephalob_skims_basecalling_fast.bam > Cephalob_skims_basecalling_fast.fastq
```
5. Check data quality with the fast basecalling 

```=shell!
NanoPlot --fastq Cephalob_skims_basecalling_fast.fastq 
```
5. Demultiplex according to tags

```=shell!
guppy_barcoder -i fastq -s demultiplexed_output/ --barcode_kits "SQK-RPB114-24"
```
7. Run flye assembly for each of the barcodes

```=shell!
for fastq_file in */barcode*.fastq; do
    # Get the base name of the fastq file to create a unique output folder for each assembly
    base_name=$(basename "$fastq_file" .fastq)
    
    # Create a directory for the Flye output
    mkdir -p "$base_name"
    
    # Run Flye on the FASTQ file (using the full path to the Flye executable)
   flye --nano-raw "$fastq_file" --out-dir "$base_name" --threads 8
    
    # Optionally, check if Flye finished without errors (exit code 0)
    if [ $? -eq 0 ]; then
        echo "Assembly for $base_name completed successfully!"
    else
        echo "Assembly for $base_name failed!"
    fi
done
```
8. Rename each fly assembly according to barcode number and move all assemblies to new foler. 

```=shell!
# Define the destination directory
DEST_DIR="./drafts_flye_fasbasecalling"

# Create the destination directory if it doesn't exist
mkdir -p "$DEST_DIR"

# Loop through each barcode folder
for folder in ./barcode*; do
    if [ -d "$folder" ]; then
        # Check if assembly.fasta exists
        if [ -f "$folder/assembly.fasta" ]; then
            # Extract folder name (e.g., barcodeXX)
            folder_name=$(basename "$folder")
            
            # Define the new filename
            new_filename="${folder_name}_assembly.fasta"
            
            # Copy and rename the file
            cp "$folder/assembly.fasta" "$DEST_DIR/$new_filename"
            
            echo "Copied and renamed: $folder/assembly.fasta -> $DEST_DIR/$new_filename"
        else
            echo "assembly.fasta not found in $folder"
        fi
    fi
done
```

9. Run busco on all assemblies and store results per assembly in individual folders. Docker is used since this was run on Julio.

```=shell!
# Directory containing the assemblies
ASSEMBLY_DIR="./drafts_flye_fasbasecalling"

# Loop through each assembly file
for assembly in "$ASSEMBLY_DIR"/*.fasta; do
    if [ -f "$assembly" ]; then
        # Extract the base name of the assembly file (e.g., barcodeXX_assembly)
        assembly_name=$(basename "$assembly" .fasta)

        # Create an output directory for this assembly
        output_dir="./busco_results_${assembly_name}"
        mkdir -p "$output_dir"

        # Run BUSCO
        docker run -u $(id -u) -v $(pwd):/busco_wd ezlabgva/busco:v5.4.7_cv1 busco \
            -i "/busco_wd/$assembly" \
            -o "$assembly_name" \
            -l nematoda_odb10 \
            -m genome \
            -c 20 \
            --out_path "/busco_wd/$output_dir"

        echo "BUSCO analysis completed for $assembly"
    else
        echo "No .fasta files found in $ASSEMBLY_DIR"
    fi
done
```

10. Move all summary results to anew folder. 

```=shell!
# Define the directory where summary files will be collected
SUMMARY_DIR="./all_summaries"
mkdir -p "$SUMMARY_DIR"

# Find and move all summary files to the new directory
for summary_file in ./busco_results_*/**/short_summary*.txt; do
    if [ -f "$summary_file" ]; then
        # Get the base name of the summary file
        base_name=$(basename "$summary_file")

        # Move the file to the summaries directory
        mv "$summary_file" "$SUMMARY_DIR/$base_name"

        echo "Moved: $summary_file -> $SUMMARY_DIR/$base_name"
    else
        echo "No summary files found."
    fi
done
```

