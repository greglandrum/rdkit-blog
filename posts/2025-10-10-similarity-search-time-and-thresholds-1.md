---
categories:
- cartridge
- questions
date: '2025-10-10'
layout: post
title: Similarity search time and thresholds
description: How does the similarity threshold affect search times using the cartridge?
toc: true

---

This is an updated and modified version of a [post I wrote back in 2015](https://rdkit.blogspot.com/2015/08/impact-of-threshold-on-similarity.html).
# Impact of the threshold on similarity search times

A
[ChEMBL blog post](http://chembl.blogspot.ch/2015/08/lsh-based-similarity-search-in-mongodb.html)
included the (to me) somewhat surprising result that similarity search
times while using the RDKit cartridge did not show much of a
dependency on the similarity threshold being used. I wanted to
investigate this a bit more closely.

I'm using a local install of ChEMBL35 for these tests. Since I don't
have the patience to wait for searches to complete with 1000
molecules, I will randomly select just 10:

    chembl_35=# select * into temporary table foo from rdk.fps order by random() limit 10;
    SELECT 10
    chembl_35=# select molregno,m from rdk.mols join foo using (molregno);
    molregno |                                                      m                                                      
    ----------+-------------------------------------------------------------------------------------------------------------
        4111 | C=C1CC2(CCCCCCCCC2)OC1=O
        4131 | CC1Cc2cc3c(cc2C(c2ccc(N)cc2)=NN1C=O)OCO3
        4185 | CCOC(=O)/C=C(C)/C(F)=C/C=C(C)/C=C/c1c(C)cc(OC)c(C)c1C
        4256 | CC(C)(C)c1cc(CCc2cccnc2)cc(C(C)(C)C)c1O
        4292 | O=C(CCCCCCCCCBr)CC(=O)N[C@H]1CCOC1=O
        4334 | CCCC[C@@H](C[C@@H](CCc1ccc(-c2ccc(F)cc2)cc1)C(=O)N[C@H](C(=O)Nc1ccccc1)C(C)(C)C)C(=O)O
        4335 | COc1ccc(/C=C/c2cc(C(C)(C)C)c(O)c(C(C)(C)C)c2)cc1
        4389 | Cc1ccc2c(c1)C(=O)N(CC(C)(C)C[N+](C)(C)CCCCCC[N+](C)(C)CC(C)(C)CN1C(=O)c3cccc4cccc(c34)C1=O)C2=O.[Br-].[Br-]
        4591 | O=C(NCCCn1ccnc1)c1cc2ccccc2[nH]1
        4599 | O=Cc1ccn(-c2cc3c(cc2Cl)nc(O)n2nc(C(=O)O)cc32)c1
    (10 rows)

And now look at timing results for a number of threshold values:

    chembl_35=# set rdkit.tanimoto_threshold=0.4;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.202 ms
    count 
    -------
    3431
    (1 row)

    Time: 2446.118 ms (00:02.446)
    chembl_35=# set rdkit.tanimoto_threshold=0.5;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.245 ms
    count 
    -------
    814
    (1 row)

    Time: 2258.420 ms (00:02.258)
    chembl_35=# set rdkit.tanimoto_threshold=0.6;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.146 ms
    count 
    -------
    261
    (1 row)

    Time: 2005.068 ms (00:02.005)
    chembl_35=# set rdkit.tanimoto_threshold=0.7;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.304 ms
    count 
    -------
    122
    (1 row)

    Time: 1702.164 ms (00:01.702)
    chembl_35=# set rdkit.tanimoto_threshold=0.8;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.147 ms
    count 
    -------
        35
    (1 row)

    Time: 1272.436 ms (00:01.272)
    chembl_35=# set rdkit.tanimoto_threshold=0.9;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.207 ms
    count 
    -------
        21
    (1 row)

    Time: 645.774 ms
    chembl_35=# set rdkit.tanimoto_threshold=0.95;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.101 ms
    count 
    -------
        17
    (1 row)

    Time: 268.827 ms
    chembl_35=# set rdkit.tanimoto_threshold=0.99;select count(*) from rdk.fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.242 ms
    count 
    -------
        14
    (1 row)

    Time: 54.065 ms


In the previous version of the post, there were some interesting differences here. That's no longer the case. this is probably at least partially because both Postgres and the cartridge have seen some improvements in the last ten years (for example, index-only scans are now possible). I guess the bigger part is because I'm using a more capable machine with a bunch more RAM.

## A larger test: ZINC

To see if there are any trends for a larger dataset, I grabbed a
copy of the
[ZINC All Clean set](http://zinc.docking.org/subsets/all-clean).
After removing the molecules that the RDKit doesn't like (there are
actually only 14 molecules in the full set that the RDKit rejected),
this leaves 16.4 million molecules.

For the record, here's how I built the database:

    (rdkit_build) glandrum@stoat:/scratch/RDKit_git/LocalData$ createdb zinc
    (rdkit_build) glandrum@stoat:/scratch/RDKit_git/LocalData$ psql -c 'create table raw_data (id SERIAL, smiles text, zinc_id char(12))' zinc
    CREATE TABLE
    (rdkit_build) glandrum@stoat:/scratch/RDKit_git/LocalData$ cd Zinc
    (rdkit_build) glandrum@stoat:/scratch/RDKit_git/LocalData/Zinc$ zcat zinc_all_clean.smi.gz | sed '1d; s/\\/\\\\/g' |psql -c "copy raw_data (smiles,zinc_id) from stdin with delimiter ' '" zinc
    COPY 16403864
    (rdkit_build) glandrum@stoat:/scratch/RDKit_git/LocalData/Zinc$ psql zinc
    psql (14.19 (Ubuntu 14.19-0ubuntu0.22.04.1))
    Type "help" for help.

    zinc=# \timing
    Timing is on.
    zinc=# create extension rdkit;
    CREATE EXTENSION
    Time: 21.754 ms
    zinc=# select * into mols from (select id,mol_from_smiles(smiles::cstring) m from raw_data) tmp where m is not null;
      ... snip ...
    SELECT 16403848
    Time: 945195.267 ms (15:45.195)
    zinc=# select id,morganbv_fp(m) as mfp2 into fps from mols;
    SELECT 16403848
    Time: 155185.639 ms (02:35.186)
    zinc=# create index fps_mfp2_idx on fps using gist(mfp2);
    CREATE INDEX
    Time: 140542.861 ms (02:20.543)
    zinc=# select * into temporary table foo from fps tablesample bernoulli (1) repeatable (123456) limit 10;
    SELECT 10
    Time: 11.977 ms

Here are the 10 random molecules:

    zinc=# select id,m from mols join foo using (id);
    id    |                             m                             
    ------+-----------------------------------------------------------
    12317 | CCc1nnc(NC(=O)[C@H](CC)Oc2ccccc2)s1
    385   | CCOC(=O)c1ccccc1C(=O)OCC
    24317 | CC(C)(C)c1ccc([C@@]23OC(=O)C(C)(C)[C@@H]2OC(=O)C3(C)C)cc1
    48    | CC(C)(C)[NH2+]C[C@H](O)COc1cc(Cl)ccc1Cl
    225   | CC[C@@]1(CO)CCC[NH+]2CCc3c([nH]c4ccccc34)[C@@H]21
    90    | C[NH2+]C[C@@H](OC)c1cccc(C(F)(F)F)c1
    24406 | CCOC1(OCC)[NH+]=C(N)[C@@]2(C#N)C3(CCCCC3)[C@@]12C#N
    24738 | Cc1ccc(S(=O)(=O)Nc2ccccc2C#N)cc1
    356   | CCc1c(O)c(=O)ccn1CC
    8     | O=C([O-])[C@@H](O)c1ccccc1
    (10 rows)

    Time: 5167.539 ms (00:05.168)

Now the searches:

    zinc=# set rdkit.tanimoto_threshold=0.4; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.133 ms
    count 
    -------
    55762
    (1 row)

    Time: 15931.816 ms (00:15.932)
    zinc=# set rdkit.tanimoto_threshold=0.5; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.183 ms
    count 
    -------
    4486
    (1 row)

    Time: 13616.273 ms (00:13.616)
    zinc=# set rdkit.tanimoto_threshold=0.6; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.205 ms
    count 
    -------
    623
    (1 row)

    Time: 11555.706 ms (00:11.556)
    zinc=# set rdkit.tanimoto_threshold=0.7; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.213 ms
    count 
    -------
    165
    (1 row)

    Time: 8509.171 ms (00:08.509)
    zinc=# set rdkit.tanimoto_threshold=0.8; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.097 ms
    count 
    -------
        50
    (1 row)

    Time: 4717.467 ms (00:04.717)
    zinc=# set rdkit.tanimoto_threshold=0.9; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.097 ms
    count 
    -------
        20
    (1 row)

    Time: 1026.567 ms (00:01.027)
    zinc=# set rdkit.tanimoto_threshold=0.95; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.103 ms
    count 
    -------
        20
    (1 row)

    Time: 195.971 ms
    zinc=# set rdkit.tanimoto_threshold=0.99; select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
    SET
    Time: 0.096 ms
    count 
    -------
        20
    (1 row)

    Time: 34.866 ms
 
I'm going to look at two interesting steps in this data: from a threshold of 0.4 to 0.6, and from 0.6 to 0.99. In the first case, the number of hits goes from 55762 to 623, while the time goes from 15.9s to 11.6s. In the second case, the number of hits goes from 623 to 20, while the time goes from 11.6s to 34.9ms. In the first case the number if hits drops by almost two orders of magnitude while the time only drops by about 25%. In the second case, the number of hits drops by a factor of 30, while the time drops by a factor of almost 39.


Look at the output from `EXPLAIN ANALYZE` for these queries to see if we can figure out what's going on:

`   zinc=# set rdkit.tanimoto_threshold=0.4;
    SET
    Time: 0.152 ms
    zinc=# explain (analyze on, buffers on) select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
                                                                        QUERY PLAN                                                                      
    ------------------------------------------------------------------------------------------------------------------------------------------------------
    Aggregate  (cost=1376580.94..1376580.95 rows=1 width=8) (actual time=16239.641..16239.642 rows=1 loops=1)
    Buffers: shared hit=9573 read=2374633, local hit=1
    ->  Nested Loop  (cost=0.42..1324498.62 rows=20832924 width=0) (actual time=8.908..16236.122 rows=55762 loops=1)
            Buffers: shared hit=9573 read=2374633, local hit=1
            ->  Seq Scan on foo  (cost=0.00..22.70 rows=1270 width=32) (actual time=0.004..0.016 rows=10 loops=1)
                Buffers: local hit=1
            ->  Index Only Scan using fps_mfp2_idx on fps fps1  (cost=0.42..878.85 rows=16404 width=65) (actual time=6.231..1622.310 rows=5576 loops=10)
                Index Cond: (mfp2 % foo.mfp2)
                Heap Fetches: 0
                Buffers: shared hit=9573 read=2374633
    Planning Time: 0.052 ms
    JIT:
    Functions: 5
    Options: Inlining true, Optimization true, Expressions true, Deforming true
    Timing: Generation 0.126 ms, Inlining 1.992 ms, Optimization 3.165 ms, Emission 3.565 ms, Total 8.848 ms
    Execution Time: 16239.872 ms
    (16 rows)

    Time: 16240.250 ms (00:16.240)
    zinc=# set rdkit.tanimoto_threshold=0.6;
    SET
    Time: 0.114 ms
    zinc=# explain (analyze on, buffers on) select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
                                                                        QUERY PLAN                                                                      
    ------------------------------------------------------------------------------------------------------------------------------------------------------
    Aggregate  (cost=1376580.94..1376580.95 rows=1 width=8) (actual time=13299.969..13299.970 rows=1 loops=1)
    Buffers: shared hit=4685 read=1995775, local hit=1
    ->  Nested Loop  (cost=0.42..1324498.62 rows=20832924 width=0) (actual time=112.152..13299.772 rows=758 loops=1)
            Buffers: shared hit=4685 read=1995775, local hit=1
            ->  Seq Scan on foo  (cost=0.00..22.70 rows=1270 width=32) (actual time=0.004..0.013 rows=10 loops=1)
                Buffers: local hit=1
            ->  Index Only Scan using fps_mfp2_idx on fps fps1  (cost=0.42..878.85 rows=16404 width=65) (actual time=112.027..1329.058 rows=76 loops=10)
                Index Cond: (mfp2 % foo.mfp2)
                Heap Fetches: 0
                Buffers: shared hit=4685 read=1995775
    Planning Time: 0.052 ms
    JIT:
    Functions: 5
    Options: Inlining true, Optimization true, Expressions true, Deforming true
    Timing: Generation 0.126 ms, Inlining 2.130 ms, Optimization 3.170 ms, Emission 3.601 ms, Total 9.027 ms
    Execution Time: 13300.146 ms
    (16 rows)

    Time: 13300.681 ms (00:13.301)
    zinc=# set rdkit.tanimoto_threshold=0.99;
    SET
    Time: 0.253 ms
    zinc=# explain (analyze on, buffers on) select count(*) from fps fps1 cross join foo where fps1.mfp2%foo.mfp2;
                                                                    QUERY PLAN                                                                   
    ------------------------------------------------------------------------------------------------------------------------------------------------
    Aggregate  (cost=1376580.94..1376580.95 rows=1 width=8) (actual time=50.800..50.801 rows=1 loops=1)
    Buffers: shared hit=14558 read=5872, local hit=1
    ->  Nested Loop  (cost=0.42..1324498.62 rows=20832924 width=0) (actual time=11.722..50.787 rows=12 loops=1)
            Buffers: shared hit=14558 read=5872, local hit=1
            ->  Seq Scan on foo  (cost=0.00..22.70 rows=1270 width=32) (actual time=0.003..0.010 rows=10 loops=1)
                Buffers: local hit=1
            ->  Index Only Scan using fps_mfp2_idx on fps fps1  (cost=0.42..878.85 rows=16404 width=65) (actual time=2.392..4.188 rows=1 loops=10)
                Index Cond: (mfp2 % foo.mfp2)
                Heap Fetches: 0
                Buffers: shared hit=14558 read=5872
    Planning Time: 0.052 ms
    JIT:
    Functions: 5
    Options: Inlining true, Optimization true, Expressions true, Deforming true
    Timing: Generation 0.123 ms, Inlining 2.134 ms, Optimization 3.203 ms, Emission 3.554 ms, Total 9.014 ms
    Execution Time: 50.972 ms
    (16 rows)

    Time: 51.323 ms
`
The big difference I see here is the number of buffers that actually need to be read when executing the queries. At the 0.4 threshold, we need to read 2.37 million buffers, at 0.7 it's just under 2 million, and at 0.99 we only need to read 5872. This is presumably because the index is pruning so many rows at the higher thresholds. I'm definitely not an expert at interpreting this output, so if anyone has other ideas or explanations, please let me know!


## An aside: chemfp performance

The previous version of this post included a performance comparison with [chemfp](http://chemfp.com/). Since I no longer have access to the licensed version of chemfp, I'm going to skip that here. I assume it's dramatically quicker than the cartridge. :-)
