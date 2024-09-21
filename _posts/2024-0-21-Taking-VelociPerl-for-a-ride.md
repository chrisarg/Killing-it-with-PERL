---
title: " Taking VelociPerl for a ride "
date: 2024-09-21
---

[VelociPerl](https://velociperl.com/) is a closed source fork of Perl that claims performance gains of 45% over the stock ("your dad's Perl" in their parlance) based on some public [benchmarks](https://github.com/ruben-work/perl-workload).
I will not go into how they achieved this performance boost, or why they released it as closed source, or even "but why the heck did you release it as closed source?", as you can follow the [Reddit discussion](https://www.reddit.com/r/perl/comments/1aft377/velociperl_a_speed_oriented_fork_of_perl/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button).
However, even a modest speed gain may be useful in some applications, so I decided to dive in a a little bit deeper.

Some of the benchmarks are numerical e.g. a linear system solver, generating random numbers but others are more relevant to garden variety Perl tasks, e.g. generating objects as blessed hashes. 
These tasks may appear in the context of some applications, so it is nice to know there are some benefits, but why not come up with a composite task and benchmark it? My usage of Perl involves random number generation, operations on tasks, creation of objects, string concatenation and function calling, so I figured there should be a way to combine all of them together and take VelociPerl for a ride.

Consider the following code that executes two benchmarks:
* Generating a Hash (in which the keys are random strings and the values are obtained as a blessed reference to an anonymous scalar)
* Accessing the said hash

```perl
use v5.38;
use Benchmark qw(timethis);    # for benchmarking

my $Length_of_accesstest = 1_000_000;
my $Length_of_gentest=100_000;
my $class = 'HashingAround';    
sub generate_random_string(@alphabet) {
    my $len = scalar @alphabet;
    my $length = int( rand(100) ) + 1;    # random length between 1 and 100
    return join '', @alphabet[ map { rand  $len } 1 .. $length ];
}

my %hash_of_seqs;

for ( 1 .. $Length_of_accesstest ) {
    my $key = generate_random_string(('A' .. 'Z'));
    my $val = bless \do { my $anon_scalar = $key }, $class;
    $hash_of_seqs{$key} = $val;
}

say "=" x 80;
say "\tTiming hash access using timethis";
timethis(
    20,
    sub {
        while ( my ( $seq_id, $seq ) = each %hash_of_seqs ) {
            ## worthless access to seq to force the sequence to be copied in memory
            my $sequence = $seq;
        }
    }
);

say "=" x 80;
say "\tTiming hash generation using timethis";
timethis(
    10,
    sub {
        my %hash_of_seqs;
        for ( 1 .. $Length_of_gentest ) {
    my $key = generate_random_string(('A' .. 'Z'));
    my $val = bless \do { my $anon_scalar = $key }, $class;
            $hash_of_seqs{$key} = $val;
        }
    }
);

```

To switch the Perl interpreter one simply changes the shebang line. In my oldish dual Xeon E5-2697 v4, I obtained the following results

* **Stock Perl** :
```text
(base) chrisarg@chrisarg-HP-Z840-Workstation:~/software-dev/velociperlHash$ ./testHash_speed.pl 
================================================================================
	Timing hash access using timethis
timethis 20: 12 wallclock secs (11.61 usr +  0.00 sys = 11.61 CPU) @  1.72/s (n=20)
================================================================================
	Timing hash generation using timethis
timethis 10: 12 wallclock secs (12.58 usr +  0.01 sys = 12.59 CPU) @  0.79/s (n=10)

```

* **VelociPerl** :
```text
(base) chrisarg@chrisarg-HP-Z840-Workstation:~/software-dev/velociperlHash$ ./testHash_speed_vperl.pl 
================================================================================
	Timing hash access using timethis
timethis 20: 11 wallclock secs (11.04 usr +  0.00 sys = 11.04 CPU) @  1.81/s (n=20)
================================================================================
	Timing hash generation using timethis
timethis 10: 10 wallclock secs ( 9.80 usr +  0.01 sys =  9.81 CPU) @  1.02/s (n=10)
```

Walking the hash was not materially different between the two interpreters, but the more compute intense task that involved random numbers, string operation and blessing of objects was ~30% faster. 
This gain may or may not justify using a closed source version of Perl to you. But it is worth noting that one _may_ be able to tweak the compilation of the Perl source (which appears to be a major source of the claimed gains) to generate faster executing Perl code. Perhaps an approach that one can try with an open sourced fork?
