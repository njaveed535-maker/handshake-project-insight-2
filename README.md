# handshake-project-insight-2
Evaluating scientific reasoning and biological evidence boundaries in open-weight language models.
Ambient RNA Correction Without Erasing Rare Cells
1. Why is this task genuinely difficult?
The professional and why it matters. This is work done by a computational biologist or bioinformatician in a genomics core facility or single-cell research lab — typically someone with a PhD in molecular biology, genetics, or a related field plus several years of hands-on scRNA-seq analysis. In the real world, ambient RNA contamination is one of the most common and insidious artifacts in droplet-based single-cell data: cell-free mRNA from lysed cells is captured in droplets alongside intact cells, inflating counts and creating spurious expression signatures that can be mistaken for real biology . A professional is paid to correct this without destroying genuine signal — a judgment call that directly affects downstream discoveries about rare cell types, continuous developmental trajectories, and subtle disease-associated states.

Synthetic data, realistically complex. The data is synthetic with hidden ground truth, generated to embed the complexity a real multi-batch experiment would have: (i) three batches with batch-specific ambient profiles; (ii) gene-specific ambient contributions, so highly expressed genes dominate the soup; (iii) contamination correlated with cell size and cell type, so a single global contamination fraction is provably wrong; (iv) one batch where empty droplets are scarce, so the ambient profile must be partly inferred from the cells themselves; (v) a capture-bias term, so the ambient profile is not simply the pooled empty-droplet profile; and (vi) a rare subpopulation whose counts are roughly 60% ambient, making it nearly indistinguishable from soup under naive methods.

Key reasoning and methodological pitfalls. The core challenge is that ambient RNA correction is not denoising — it is attribution: separating, for each cell and each gene, how many observed counts came from the cell's true transcriptome versus the ambient pool. The pitfalls that produce wrong answers are:

Over-correction erases rare cells and flattens continuous states — the correction removes real biology along with the contamination .

Under-correction leaves contamination that creates false co-expression patterns and spurious cell-type assignments.

A single global contamination fraction fails on at least one batch, because ambient profiles differ by batch.

Uniform gene treatment over-corrects lowly expressed genes and under-corrects highly expressed ones.

Confusing ambient RNA with doublets — a different artifact with a different signature and a different correction.

Producing an invalid count matrix — negative or fractional counts, or corrected counts exceeding observed counts .

Treating the ambient profile as the pooled empty-droplet profile — fails because of size-dependent capture bias.

Running published tools out of the box (SoupX, DecontX, CellBender defaults) — fails because the generator deliberately violates their core assumptions: batch-uniform profile, empty-droplet sufficiency, and size-independent capture .

Why an outstanding undergraduate with unlimited time and full internet cannot get there. They can learn the math of matrix decomposition and read about the existing tools. They will not know: how to judge whether a rare population is being erased; how to set correction thresholds that preserve continuous trajectories; how to diagnose batch-specific contamination and capture bias from the data alone; or how to balance gene-specific correction across expression levels. That judgment comes from years of working with real datasets and from having seen both over- and under-correction fail in practice.

2. Intended solution approach and expert effort
High-level strategy.

Estimate per-batch ambient profiles from empty droplets, correcting for capture bias. Do not simply pool empty droplets — fit a size-dependent capture model by regressing empty-droplet gene fractions on total counts, then extrapolate to the called-cell size range.

For the batch with scarce empty droplets, partially infer the ambient profile from the cells via a joint model (EM with a hierarchical prior over batch ambient profiles).

Model each cell's observed counts as a multinomial mixture of its latent true expression and the batch ambient profile, with a cell-specific contamination fraction that can depend on cell size and inferred type.

Fit by EM (or constrained regression), enforcing non-negativity and the integer/upper-bound constraints.

Project back to a valid integer count matrix, rounding under the constraint that corrected counts are between zero and the observed counts.

Validate preservation of rare-cell marker genes and continuous trajectories before emitting the output files.

Key insight. The correction must be gene-specific, cell-specific, and batch-specific, and the ambient profile must be capture-bias-corrected. A single global contamination fraction and a pooled empty-droplet profile both fail by construction.

Effort for a qualified expert: approximately 8–14 hours. The lower bound assumes the solver has implemented a per-cell multinomial EM before and only needs to add the capture-bias regression and the scarce-empty-droplet handling. The upper bound includes debugging the integer projection and validating rare-cell preservation.

3. How the solution will be verified
What the agent must produce. Four files under /app/results/:

corrected_matrix.mtx — Matrix Market, integer, genes × cells, matching the standard 10x convention, with all entries non-negative integers no larger than the corresponding observed counts.

corrected_genes.tsv — one column, no header, gene identifiers in the same order as the input gene list.

corrected_barcodes.tsv — one column, no header, barcodes in the same order as the called-cell list.

contamination_per_cell.csv — header exactly barcode,contamination_fraction, one row per called cell in the same order as the called-cell list, with the contamination reported as a fraction in [0, 1], not a percentage.

How correctness is evaluated. The verifier reads only /app/results/* and verifier-owned hidden ground truth, and applies:

Structural gates (pass/fail): all four files exist; barcodes and genes match the input lists in exact order; matrix dimensions match; every entry is a non-negative integer no larger than the observed count; the contamination CSV header and fraction range are correct.

Contamination accuracy: mean absolute error of the per-cell contamination fraction against the hidden true value, and Spearman rank correlation overall and within every batch.

Correction accuracy: per-gene Pearson correlation between corrected and true counts, taken at the 10th percentile across genes with meaningful true expression.

Rare-cell preservation: the fraction of true rare-cell marker genes whose corrected mean expression in the rare population is at least 60% of the true mean.

Trajectory preservation: Spearman correlation between pseudotime computed on the corrected matrix and on the true matrix, using the same method on both sides.

Anti-gaming checks: the submission must materially reduce ambient-marker expression in cells of the wrong type; it must strictly beat the identity (no-correction) baseline on correction accuracy and ambient-marker drop; and it must strictly beat a deliberate over-correction baseline on rare-cell retention.

Why the band distinguishes correct from incorrect solutions. Thresholds were calibrated by simulating the generative model and running four baselines: no correction, a global-fraction method without capture correction, two structurally different correct methods (per-cell multinomial EM and constrained regression, both with capture-corrected profiles), and deliberate over-correction. The accepted band sits strictly between the correct methods and every failing baseline, with at least two standard deviations of headroom on each side. No threshold sits within one standard deviation of either a passing or a failing baseline. The band therefore admits genuine variation across sound methods — different EM initializations, different rounding strategies, different capture-model extrapolations — while excluding no-op, global-fraction, and over-correction submissions.

Multiple valid answers. The verifier does not require a specific algorithm. Any method that produces the four files with the correct schema and clears every gate passes. Two structurally different correct methods were simulated and both clear the band with margin.

Ground truth cannot be looked up. The hidden ground truth — true counts, true per-cell contamination fractions, true ambient profiles, rare-cell labels, trajectory order — is generator-internal and never present in /app/data or in any container artifact the agent can read. It is overlaid only at verify time. No published dataset matching this generator exists, so the answer cannot be memorized or hardcoded.

4. Category and sub-category justification
Category: Science

Sub-Category: Biology

This task belongs to Science because it requires frontier domain knowledge in biology — specifically, the biology of droplet-based single-cell RNA sequencing, ambient RNA contamination, and the trade-offs involved in correcting it without erasing rare or continuous biological states. It is not a generic matrix-decomposition or data-cleaning exercise: solving it requires understanding what ambient RNA is, how it arises, why it differs by batch, why size-dependent capture bias decouples the ambient profile from pooled empty droplets, and why over-correction is biologically destructive. The data, the problem, and the verification are all rooted in the real-world practice of scRNA-seq analysis. The professional who would perform this work is a computational biologist or bioinformatician in a genomics core facility, and the value of the work — preserving rare cell types and continuous trajectories for downstream discovery — is specific to single-cell genomics. The sub-category is therefore a coherent child of Biology, and the task would be recognized by any practicing single-cell analyst as genuine, paid professional work.

