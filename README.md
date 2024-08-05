## IsAb2.0

<p align="center">
<img width="450" src="https://github.com/zeysun/IsAb2.0/blob/main/Workflow.jpg">
</p>

## Workflow of IsAb2.0
The general procedure of IsAb2.0 is outlined in Figure 1. In the first step, users input sequences of the antibody and antigen into the protocol, with antibody sequences retrievable from the IMGT (https://www.imgt.org/) database. In step 2, AlphaFold-Multimer2.3/3.0 generates 3D structures of the antibody, antigen, and their complex, effectively combining the functions of homology modeling and global docking from IsAb1.0. The per-residue confidence metric (pLDDT) is used to evaluate the quality of the model generated by AlphaFold-Multimer, assessing local accuracy. If the pLDDT scores are below 70, the antibody-antigen complex undergoes further refinement. Otherwise, the complex proceeds to step 4 for local docking. In step 3, several methods can be used for structural refinement. If the structure with lower pLDDT score has a crystal structure, this crystal structure will be optimized by Rosetta FastRelax to resolve potential clashes and identify energetically favorable conformations. The relaxed structures will then replace the model generated by AlphaFold-Multimer. If a crystal structure is unavailable, SWISS-MODEL will generate a new 3D homology model for low pLDDT structure and replace it. In step 4, SnugDock performs local docking for the antibody and antigen, allowing flexibility of the CDR loops and interfacial side chains. SnugDock refines the potential binding poses provided by AlphaFold-Multimer and outputs the final antibody-antigen complex. In step 5, after obtaining the 3D structure of antibody-antigen complex, alanine scanning is performed to predict possible hotspots on the antibody, facilitating future antibody design. Alanine scanning mutates the interface residues to alanine and calculates the change in energy to identify hotspots. In step 6, point mutations are applied to mutate the antibody interface residues to the remaining 17 amino acids and identify mutations that can improve the binding affinity of the complex. The antibody-antigen complex is imported into FlexddG to perform point mutations, identifying mutations that enhance binding affinity.

## Antibody-antigen structural complex generation
Input the antibody and antigen sequences into COSMIC2 (https://cosmic-cryoem.org/tools/alphafold2/). In the parameters setting, set the “Number of predictions per model” to 10, the other parameters using the setting below. For more detailed instructions on how to use COSMIC2, you can find the tutorial on its website.
```
Database: full_dbs
Model: multimer
Number pf predictions per model: 10
Latest date (YYYY-mm-dd) to use for template search (if using templates): 2023-05-30
Models to relax (all,best,none): best
```

## Structure refinement and local docking of antibody-antigen complex
If the output of AlphaFold-Multimer has a pLDDT value lower than 70, users can input the sequences of the antibody or antigen to SWISS-MODEL (https://swissmodel.expasy.org/) to generate more accurate 3D structures of the antibody or antigen. For more detailed instructions of how to use SWISS-MODEL, you can find the tutorial on its website.
Before refining the starting complex with SnugDock, optimize the structures of the antibody and antigen using FastRelax. Relax the structures of the antibody and antigen to generate 10 ensembles for each of them. FastRelax code:
```
mpirun -np 8 relax.mpi.linuxgccrelease -s structure.pdb -in:file:fullatom -relax:thorough -relax:constrain_relax_to_start_coords -relax:ramp_constraints false -ex1 -ex2  -use_input_sc -flip_HNQ -no_optH false -nstruct 10
```
Repack the starting complex. Users need to create list files for antibody and antigen ensembles. The format of ensemble list file can be found in Repack folder. Repack code:
```
docking_prepack_protocol.mpi.linuxgccrelease -in:file:s starting_complex.pdb -ex1 -ex2 -partners A_H -ensemble1 antigen_ensemble.list -ensemble2 antibody_ensemble.list -docking:dock_rtmin
```

Repack step will output the repacked complex, which will serve as input to SnugDock. SnugDock code:
```
 mpirun -np 64 snugdock.mpi.linuxgccrelease -s repacked_complex.pdb -auto_generate_h3_kink_constraint -h3_loop_csts_hr -spin -dock_pert 3 8 -loops:refine_outer_cycles 2 -loops:max_inner_cycles 20 -detect_disulf false -partners A_H -out:file:scorefile score-snugdock.sf -nstruct 1000 -docking_low_res_score motif_dock_score -mh:path:scores_BB_BB path/to/database/additional_protocol_data/motif_dock/xh_16_ -mh:score:use_ss1 false -mh:score:use_ss2 false -mh:score:use_aa1 true -mh:score:use_aa2 true -value_pH 7.0
```
If three of the five lowest I_sc results have an I_rms value lower than 4 Å, it indicates that the docking is successful. Users can select the decoy with the lowest I_sc as the final result.


# Computational alanine scanning and point mutation
Possible hotspots (or key residues) on the antibody were predicted by Rosetta alanine scanning program. Among the alanine scanning results, residues with a ΔΔG higher than 1 kcal/mol were selected as hotspots on the antibody. Alanine scanning code:
```
mpirun -np 8 rosetta_scripts.mpi.linuxgccrelease -s final_complex.pdb -use_input_sc -nstruct 10 -jd2:ntrials 1 -database /mnt/d/rosetta_src_2021.16.61629_bundle_2/main/database -parser:protocol AlaScan.xml -parser:view -out:overwrite
```

If users do not have any specific residues they want to mutate, they can mutate the interface residues of the complex. The interface residues of the antibody-antigen complex need to be defined. Define interface code:
```
python define_interface.py --side1 A --side2 H --design-side 1 --repack --output final_complex final_complex.pdb
```

FlexddG is used to perform mutation screening for the interface residues. A resfile needs to be created, which contains the residue numbers (based on PDB) that users want to mutate. The format of resfile can be found in FlexddG folder. Users need to create a folder named “inputs” and create a subfolder inside it. The subfolder contains two files, the input PDB file and a “chain_to_move.txt” file that contains the letter of the chain you want to mutate. FlexddG outputs two folders, ‘analysis_output’ and ‘output_saturation’ folders. “analyze_flex_ddG.py” script can be used to analyze the results of FlexddG. “extract_structures.py” script can be used to extract mutated structures from the results. FlexddG code:
```
python run_example.py
python analyze_flex_ddG.py output_saturation
python extract_structures.py output_saturation
```
