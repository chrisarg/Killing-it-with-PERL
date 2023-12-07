---
title: "Is Perl a write only language? "
date: 2023-12-06
---

I started using Perl in the early 21st century during the late phase of my PhD and postdoc to glue together 
various components in C and do some bioinformatics work in gene expression analysis. In the late 2000s switched
to R as my research interests became more aligned with clinical dataset analysis and biostatistical applications.
In the last couple of years, as my research on discovering RNA biomarkers from biofluids (blood, urine, and other
yucky stuff), I started using a combination of the (excellent) Bioconductor R facilities and considered Python for
the construction of applications regarding the analysis of my own (and other people's! biological sequencing data).
But something did not seem right ... so I looked back to my past. Could Perl be useful again? Despite many saying
it is a dead, read only language, the Perl of the 2020s is beautiful, feature rich and useful. 
Judge for your self if the following code (which downloads a bunch of sequence files from the ENSEMBL genome database
project, computes some basic statistics about the length of the RNA molecules of the Homo sapiens (that would be you!) 
is a throw away piece of code that you would not be able to decipher 30 minutes after you had written it. 

#!/home/chrisarg/perl5/perlbrew/perls/current/bin/perl
use v5.38;
###############################################################################
## dependencies
use File::Basename;           # for extracting the filename from a path
use File::Spec;               # for creating cross platform paths
use FindBin qw($Bin);         # for finding the script location
use Bio::DB::Fasta;           # for indexing the fasta files
use Bio::SeqIO;               # for reading the fasta files
use LWP::Simple;              # for downloading the fasta files
use PDL;                      # for the PDL data structure
use PDL::Graphics::PLplot;    # for plotting
use PDL::NiceSlice;           # nice slice syntax
use PDL::Stats qw(:all);
use PDL::Ufunc;               # for the PDL statistics
###############################################################################
## set up the download directory
our $download_dir = File::Spec->catfile( $Bin, 'fastaloc' );
mkdir $download_dir unless -e $download_dir;
our $db_stem = 'HS_cds_ncRNA';

print <<"DOWNLOAD";
## The human transcriptome files at ENSEMBL consists of three main collections:
## CDS (22.7MB),  ncRNA (19MB), CDNA (75MB) ensembl fasta gz files
## which uncompress to: 
## CDS (179.0MB),  ncRNA (96.4MB), cDNA (449.5MB) ensembl fasta files
## Note that the seq ids in the CDS file are duplicated in the cdna file
## which also includes UTR and introns, not just the protein coding sequence as 
## the CDS file does. For this script we will use the ncRNA and cDNA files.
DOWNLOAD

## download the fasta files and uncompress if those do not exist
our @fasta_files = qw(
  https://ftp.ensembl.org/pub/release-110/fasta/homo_sapiens/ncrna/Homo_sapiens.GRCh38.ncrna.fa.gz
  https://ftp.ensembl.org/pub/release-110/fasta/homo_sapiens/cdna/Homo_sapiens.GRCh38.cdna.all.fa.gz
);
our @index_files;
for my $dataset (@fasta_files) {
    ## download the fasta files into $download_dir and upack them
    my $dataset_fname             = basename($dataset);
    my $uncompressed_dataset_name = $dataset_fname =~ s/.gz//r;
    $dataset_fname = File::Spec->catfile( $download_dir, $dataset_fname );
    $uncompressed_dataset_name =
      File::Spec->catfile( $download_dir, $uncompressed_dataset_name );
    ## skip those already downloaded (e.g. from a previous run)
    unless ( -e $dataset_fname || -e $uncompressed_dataset_name ) {
        my $rc = getstore( $dataset, $dataset_fname );
        if ( is_error($rc) ) {
            next "getstore of <$dataset> failed with $rc";
        }
    }
    system 'gzip', '-d', $dataset_fname
      unless -e $uncompressed_dataset_name;
    push @index_files, $uncompressed_dataset_name;
}
###############################################################################
## PDL fun - use perl's dynamic arrays to extract the transcript lengths
## and then use PDL to calculate the summary statistics and plot the histogram
my @human_transcript_length = ();

for my $file (@index_files) {
    my $in = Bio::SeqIO->new(
        -file   => $file,
        -format => 'Fasta'
    );
    while ( my $seq = $in->next_seq ) {
        push @human_transcript_length, length $seq->seq;
    }
}
my $seq_length = pdl(@human_transcript_length);

## calculate the summary statistics
sub summary_stats($data) {   ## Perl has prototypes!
    unless ( $data->isa('PDL') ) {
        warn "summary_stats expects a PDL object";
        return undef;
    }
    my $mean       = avg($data);
    my $median     = median($data);
    my $std        = stdv_unbiased($data);
    my $min        = min($data);
    my $max        = max($data);
    return ( $mean, $median, $std, $min, $max );
}


say "\n", "-" x 80;
printf("Summary Stats for Human Transcript Lengths\n");
printf( "Number of transcripts: %15d\tMean : %8.2f\tMedian : %8.2f\n"
      . "Std Dev : %8.2f\t, Min: %8d\tMax  : %8d\n",
    $seq_length->nelem,
    summary_stats($seq_length) ); ## list interpolation of all stats!

say "-" x 80;
my $log10_seq_length = log10 $seq_length;

## plot a histogram of the transcript lengths

my $pl = PDL::Graphics::PLplot->new( DEV => 'png', FILE => 'test.png' );
$pl->histogram1(
    $log10_seq_length, 500,
    TITLE => 'Human Transcript Lengths',
    XLAB  => 'log10(Transcript Length)',
    YLAB  => 'Frequency'
);
$pl->close;


