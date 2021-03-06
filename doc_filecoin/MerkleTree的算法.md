# MerkleTree的算法

## Filecoin的算法

### create_base_merkle_tree 定义leaf拆分方法f，f以闭包函数访问数据data，因此data不需要作为参数传递到下一层调用。
  |-> from_par_iter/from_par_iter_with_config 从原始数据生成merkle tree，调用f完成数据拆分。根据原始数据size预先计算好tree height，供验证使用。
        |-> populate_data_par 将原始数据填充到叶子节点，注意，此处并未做hash，只搬运原数据
              |-> iter.opt_len  用32 bytes node迭代器计算出叶子数
              |-> iter.chunks(BUILD_DATA_BLOCK_SIZE) iter为32 bytes node迭代器，BUILD_DATA_BLOCK_SIZE为64 * BUILD_CHUNK_NODES，每次迭代处理8M数据，多个迭代可以并行填充
        |-> S::build 按层生成merkle tree，叶子节点为原始数据
              |-> process_layer 生成一层节点
                    |-> Vec::from_iter 并行计算该层merkle tree节点hash值，并将结果保存到上层创建的buffer的层开始位置
                    |-> chunk_size 节点叶子数据size总和
                    |-> chunk_nodes 节点叶子数组
                    |-> Sha256Domain::multi_node 计算hash(将叶子节点合并计算hash)
                    |-> part.hash 填充hash源数据
                          |-> sha2ni::Engine256::input
                    |-> self.hash 为填充数据计算hash
                          |-> self.0.clone
                          |-> c.result
                                |-> sha2ni::Engine256::finish 计算hash
                          |-> r.as_ref 提取hash结果
                          |-> h.copy_from_slice(ra) copy hash结果
                    |-> 结论：merkle tree的buffer开始部分数据与原始数据相同
                    |-> 问题：为何不将外部buffer传递到create_base_merkle_tree以节省32GiB内存？

### StackedBucketGraph new_stacked 创建SDR graph
  |-> 带入随机数seed，防止使用固定graph关系
  |-> 每张graph内依赖关系是固定的
  |-> base_degree
  |-> expansion_degree
  |-> feistel::precompute -> feistel_precomputed
  |-> if use_cache -> parent_cache 并行创建每层parents cache
        |-> cache创建parent消耗：219290 ms；创建一层labels：26065 ms
        |-> cache创建一层labels消耗: 6922820 ms
        |-> 无cache单层label创建消耗：7733041 ms
        |-> 11层SDR P1阶段节省：811 * 11 / 60 = 148 Min(E5 2682V4 2.5GHz)
  |-> 问题：能不能使用固定column关系，加快labels的计算?
  |-> new_seed是一个本地seed，在SDR算法上可能存在可利用空间，有可能可以事先生成很多seed并生成parents_cache，待研究?

### create_label 生成一层SDR label
  |-> 第一个32 bytes node：sha256.finish()结果放入layer_labels第一个32 bytes
  |-> 第二个之后 graph.copy_parents_data -> copy_parents_data_inner / copy_parents_data_inner_exp
        |-> 第一层以全0作为输入，第二层后以上一层结果作为输入
        |-> 从node计算parents，或者从cache获取
        |-> read_node 读取parents对应node layer label，第一层只读取base degree即6个parents node, copy 6组，第二层后读取全部，copy 3组，包括base和expansion
        |-> hasher.finish_with 将parents node数据与node数据一起hash

### add_piece 添加piece到sector
  |-> Fr32Reader { data: u64 } -> Deref后data地址作为buffer使用
  |-> sum_piece_bytes_with_alignment 获取已经写入的bytes
  |-> get_piece_alignment
        |-> MINIMUM_RESERVED_BYTES_FOR_PIECE_IN_FULLY_ALIGNED_SECTOR 127 bytes MIN_PIECE_SIZE
        |-> piece alignment size是MIN_PIECE_SIZE的2的整数次方倍
  |-> Fr32Reader::read 读取原始数据并作Fr32编码
        |-> TARGET_BITS 256, DATA_BITS 254 (可能：每256 bits中，前254 bits为有效数据，最后两个bits为0)
        |-> read_bytes 填充一次buffer并编码，target为64 bytes
              |-> full_buffer 读取8 bytes作为u64填充到buffer的data中
              |-> 每次读取8 bytes，前三个8 bytes即24 bytes原样填入target
              |-> 最后一个8 bytes按照如下规则填入target
                    |-> 4 bytes, 2 bytes, 1 bytes分三次直接填入
                    |-> 最后1 byte填入00xxxxxx，其余byte依次后延，每64 bytes延续以上规则
  |-> 对fr32_reader读取结果每64 bytes计算hash (commitment的叶子节点)
  |-> 将commitment从叶子节点生成merkle tree
  |-> PieceInfo: comm为merkle root，n为padded write bytes，written为原始piece size

### seal_pre_commit_phase1 生成SDR
  |-> compound_proof::SetupParams
        |-> vanilla_params
              |-> nodes: sector_size / 32
              |-> degree: base_degree(6)
              |-> expansion_degree: 8
              |-> seed: DRG_SEED 固定值
              |-> layer_challenges：layers和max_count，layers为SDR层数，max_count为challenges count
        |-> partitions：来自porep_config的PorepProofsPartitions，从POREP_PARTITIONS获取
  |-> CompoundProof::setup -> PublicParams
        |-> S::setup (vanilla/proof_scheme.rs) 创建StackedDRG
  |-> Build merkle tree for Sector original data：config - merkle tree存储结构，comm_d - merkle root
        |-> create_base_merkle_tree 创建merkle tree，存储到cache下面的tree_d文件
  |-> verify_pieces 使用各个piece的merkle root作为输入计算sector的merkle root，与create_base_merkle_tree中的merkle root比较
  |-> replicate_phase1
        |-> generate_labels
              |-> create_label / create_label_exp 生成一层label
  |-> Output: labels, labels config, comm_d

### seal_pre_commit_phase2 生成column hash
  |-> 从replica_path载入原始数据
  |-> 从tree_d载入原始数据merkle tree
  |-> compound_proof::SetupParams
  |-> CompoundProof::setup -> PublicParams
  |-> replicate_phase2
        |-> transform_and_replicate_layers_inner
              |-> 由于8叉树缘故，扇区大小必须能够整除8^n
              |-> 创建tree_d_config, tree_r_last_config, tree_c_config
              |-> 加载SDR labels
              |-> generate_tree_c 生成tree_c, 可使用CPU或GPU
                    |-> generate_tree_c_gpu / generate_tree_c_cpu
                          |-> rayon 并行计算column hash，此处为Poseidon hash (neptune/src/poseidon.rs)
                                |-> hash_in_mode
                          |-> MerkleTree::from_par_iter_with_config 从column hash结果生成merkle tree
                              此处merkle tree使用Poseidon hash (storage-proofs/core/src/hasher/poseidon.rs)
                          |-> 存储tree_c
              |-> 加载或计算扇区数据merkle tree，CPU并行
              |-> split_config_and_replica 为tree_r_last准备config
              |-> labels.labels_for_last_layer 读取last layer labels, elem_len=32
              |-> iter.zip 方法将last layer label和data elem按照32 bytes node组成tuple，结果为(key, data_node_bytes)
              |-> data_node_bytes计算hash，结果为data_node, 此处使用Poseidon hash
              |-> data_node与key一起encode，结果为encoded_data
                    |-> encode storage-proofs/porep/src/encode.rs
                          |-> 将key和data_node做Fr转换 此处Fr转换与Fr32编码不一样，不知细节～～
                          |-> key与data_node当作整数相加，结果为result result.add_asign和result.into
                          |-> 上两步都是在Poseidon hash中做计算，调用fff的实现，细节待研究
              |-> trees.push 从encoded_data生成tree_r_last，可能有多个config生成多棵8叉树，此处使用Poseidon hash，只能使用CPU
              |-> create_lc_tree 生成tree_r_last，将多棵8叉树生成扇区的tree_r_last
              |-> comm_r = H(comm_c || comm_r_last) Poseidon::hash2 fr.into再次出现
            　|-> 输出 tau {comm_d, comm_r}, p_aux {comm_c, comm_r_last}, t_aux {labels_config, tree_d_config, tree_r_last_config, tree_c_config}
  |-> 输出 SealPreCommitOutput {comm_r, comm_d}
  |-> 待研究：相同的hash过程使用不同hash算法的实现机制
  |-> 问题: get_base_tree_count / tree_count > 1的场景是什么?

### seal_commit_phase1 验证全部partition的merkle tree
  |-> verify_pieces 验证merkle tree
  |-> generate_replica_id 从prover_id, sector_id, ticker, comm_d创建replica_id
  |-> PublicInputs {replica_id, tau {comm_d, comm_r}, seed}
  |-> PrivateInputs {t_aux, p_aux}
  |-> compound_proof::SetupParams
  |-> CompoundProof::setup
  |-> StackedDrg::prove_all_partitions 创建vanilla_proofs
        |-> prove_layers
              |-> get_drg_parents_columns 获取x位置的parents位置，此处只获取base_degree
                    |-> graph.base_parents
                    |-> columns.push
              |-> get_exp_parents_columns 获取x位置的parents位置，只获取expansion degree
                    |-> graph.expanded_parents
            　|-> 分别prove各partitions
            　　　  |-> pub_inputs.challenges 获取challenges位置
                          |-> derive_internal 生成challenge位置
                          　　  |-> j = challenges_count * k + i i为当前challenge计数, k为partition id
                          　　  |-> hash = Sha256 replica_id, seed, j
                                |-> challenge = BigUint(hash) % (leaves - 1) + 1
                                |-> 待研究：唯一的随机量seed怎么来的？
                    |-> 针对每个challenge生成证明
                          |-> t_aux,tree_d.gen_proof 从t_aux生成challenge位置的comm_d_proof
                                |-> MerkleTree::gen_proof
                                |-> MerkleProof::try_from_proof
                          |-> 获取当前challenge位置的c_x，drg_parents，exp_parents
                          　　  |-> c_x = t_aux.column(challenge).into_proof(tree_c) column创建整列labels
                                      |-> tree_c.gen_proof
                                      |-> ColumnProof::from_column
                          |-> comm_r_last_proof = t_aux.tree_r_last.gen_cached_proof
                          |-> 待研究：特定challenge位置的proof构建
              |-> 输出：all_partition_proofs　针对所有partition的proof
  |-> verify_all_partitions verify从challenge位置数据构建的all_partition_proofs
        |-> 比较tau中的comm_r与从proofs中计算的comm_r是否匹配
        |-> proof.verify 验证proof -> 还没有看懂！！
  |-> SealCommitPhase1Output {vanilla_proofs, comm_r, comm_d, replica_id, seed, ticket}

### seal_commit_phase2 生成zk-SNARKs proofs
  |-> get_stacked_params
        |-> parameters_generator 从cache生成groth_params -> Bls12
  |-> compound_proof::SetupParams
  |-> CompoundProof::setup
  |-> StackedCompound::<Tree, DefaultPieceHasher>::circuit_proofs 生成groth_proofs
        |-> vanilla_proof 构建circuits
              |-> circuit StackedCircuit
        |-> groth16::create_random_proof_batch / groth16::create_random_proof_batch_in_priority 生成groth_proofs
              |-> create_proof_batch_priority
  |-> 将proof写入buf
  |-> verify_seal 验证封装和proof
        |-> get_stacked_verify_key 从cache加载vk
        |-> MultiProof::new_from_reader 从proof准备MultiProof
        |-> StackedCompound::verify
              |-> groth16::verify_proofs_batch
  |-> 输出 SealCommitOutput {proof: buf}

TODO:
challenge算法
groth16 proofs算法

Reference
https://eprint.iacr.org/2019/458.pdf  Starkad and Poseidon: New Hash Functions for Zero-Knowledge Proof Systems

问题：
mineOne超时看起来只做了警告，block依然被加入到了tipset

