# lotus项目编译流程

```shell

git submodule update --init --recursive
touch build/.update-modules
make -C extern/filecoin-ffi/ .install-filcrypto
make[1]: 进入目录“/home/hw/GolandProjects/lotus/extern/filecoin-ffi”
./install-filcrypto
+ auth_header=()
+ '[' -n '' ']'
++ dirname ./install-filcrypto
+ cd .
+ rust_sources_dir=rust
++ jq -r '.[].rustc_target_feature'
+ optimized_release_rustc_target_features='+adx
+sha
+sse2
+avx2
+avx
+sse4.2
+sse4.1'
++ jq -r '.[].check_cpu_for_feature | select(. != null)'
+ cpu_features_required_for_optimized_release='adx
sse2
avx2
avx
sse4_2
sse4_1'
+ main
++ get_release_type
++ local __searched=
++ local __features=
++ local __optimized=true
++ [[ ! -f /proc/cpuinfo ]]
++ __searched='cat /proc/cpuinfo | grep flags'
+++ eval 'cat /proc/cpuinfo | grep flags'
++++ cat /proc/cpuinfo
++++ grep flags
++ __features='flags            : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec'
++ for x in ${cpu_features_required_for_optimized_release[@]}
++ '[' true = true ']'
++ '[' -n '' ']'
++ for x in ${cpu_features_required_for_optimized_release[@]}
++ '[' true = true ']'
++ '[' -n '' ']'
++ for x in ${cpu_features_required_for_optimized_release[@]}
++ '[' true = true ']'
++ '[' -n '' ']'
++ for x in ${cpu_features_required_for_optimized_release[@]}
++ '[' true = true ']'
++ '[' -n '' ']'
++ for x in ${cpu_features_required_for_optimized_release[@]}
++ '[' true = true ']'
++ '[' -n '' ']'
++ for x in ${cpu_features_required_for_optimized_release[@]}
++ '[' true = true ']'
++ '[' -n '' ']'
++ '[' true == true ']'
++ '[' -n 'flags                : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single ssbd ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust sgx bmi1 avx2 smep bmi2 erms invpcid mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts pku ospke sgx_lc md_clear flush_l1d arch_capabilities
vmx flags       : vnmi preemption_timer posted_intr invvpid ept_x_only ept_ad ept_1gb flexpriority apicv tsc_offset vtpr mtf vapic ept vpid unrestricted_guest vapic_reg vid ple shadow_vmcs pml ept_mode_based_exec' ']'
++ echo '[get_release_type] configuring '\''optimized'\'' build'
[get_release_type] configuring 'optimized' build
++ echo optimized
+ local __release_type=optimized
+ '[' '' '!=' 1 ']'
+ download_release_tarball __tarball_path rust filecoin-ffi optimized
+ local __resultvar=__tarball_path
+ local __rust_sources_path=rust
+ local __repo_name=filecoin-ffi
+ local __release_type=optimized
++ git rev-parse HEAD
+ local __release_sha1=6a143e06f923f3a4f544c7a652e8b4df420a3d28
+ local __release_tag=6a143e06f923f3a4
+ local __release_tag_url=https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/tags/6a143e06f923f3a4
++ uname
+ local __release_name=filecoin-ffi-Linux-optimized
+ echo '[download_release_tarball] acquiring release @ 6a143e06f923f3a4'
[download_release_tarball] acquiring release @ 6a143e06f923f3a4
++ curl --retry 3 --location https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/tags/6a143e06f923f3a4
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  9392  100  9392    0     0  12309      0 --:--:-- --:--:-- --:--:-- 12293
+ local '__release_response={
  "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/28162207",
  "assets_url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/28162207/assets",
  "upload_url": "https://uploads.github.com/repos/filecoin-project/filecoin-ffi/releases/28162207/assets{?name,label}",
  "html_url": "https://github.com/filecoin-project/filecoin-ffi/releases/tag/6a143e06f923f3a4",
  "id": 28162207,
  "author": {
    "login": "filecoin-helper",
    "id": 37553297,
    "node_id": "MDQ6VXNlcjM3NTUzMjk3",
    "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
    "gravatar_id": "",
    "url": "https://api.github.com/users/filecoin-helper",
    "html_url": "https://github.com/filecoin-helper",
    "followers_url": "https://api.github.com/users/filecoin-helper/followers",
    "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
    "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
    "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
    "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
    "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
    "repos_url": "https://api.github.com/users/filecoin-helper/repos",
    "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
    "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
    "type": "User",
    "site_admin": false
  },
  "node_id": "MDc6UmVsZWFzZTI4MTYyMjA3",
  "tag_name": "6a143e06f923f3a4",
  "target_commitish": "6a143e06f923f3a4f544c7a652e8b4df420a3d28",
  "name": "6a143e06f923f3a4",
  "draft": false,
  "prerelease": false,
  "created_at": "2020-07-02T14:28:44Z",
  "published_at": "2020-07-02T14:35:57Z",
  "assets": [
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437790",
      "id": 22437790,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3Nzkw",
      "name": "filecoin-ffi-Darwin-optimized.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 13138992,
      "download_count": 5,
      "created_at": "2020-07-02T14:41:09Z",
      "updated_at": "2020-07-02T14:41:10Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Darwin-optimized.tar.gz"
    },
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437740",
      "id": 22437740,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3NzQw",
      "name": "filecoin-ffi-Darwin-standard.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 13138995,
      "download_count": 1282,
      "created_at": "2020-07-02T14:40:05Z",
      "updated_at": "2020-07-02T14:40:07Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Darwin-standard.tar.gz"
    },
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437646",
      "id": 22437646,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3NjQ2",
      "name": "filecoin-ffi-Linux-optimized.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 14086829,
      "download_count": 8223,
      "created_at": "2020-07-02T14:36:25Z",
      "updated_at": "2020-07-02T14:36:25Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Linux-optimized.tar.gz"
    },
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437626",
      "id": 22437626,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3NjI2",
      "name": "filecoin-ffi-Linux-standard.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 14086828,
      "download_count": 5172,
      "created_at": "2020-07-02T14:35:57Z",
      "updated_at": "2020-07-02T14:35:58Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Linux-standard.tar.gz"
    }
  ],
  "tarball_url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/tarball/6a143e06f923f3a4",
  "zipball_url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/zipball/6a143e06f923f3a4",
  "body": ""
}'
++ jq -r '.assets[] | select(.name | contains("filecoin-ffi-Linux-optimized")) | .url'
++ echo '{
  "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/28162207",
  "assets_url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/28162207/assets",
  "upload_url": "https://uploads.github.com/repos/filecoin-project/filecoin-ffi/releases/28162207/assets{?name,label}",
  "html_url": "https://github.com/filecoin-project/filecoin-ffi/releases/tag/6a143e06f923f3a4",
  "id": 28162207,
  "author": {
    "login": "filecoin-helper",
    "id": 37553297,
    "node_id": "MDQ6VXNlcjM3NTUzMjk3",
    "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
    "gravatar_id": "",
    "url": "https://api.github.com/users/filecoin-helper",
    "html_url": "https://github.com/filecoin-helper",
    "followers_url": "https://api.github.com/users/filecoin-helper/followers",
    "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
    "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
    "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
    "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
    "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
    "repos_url": "https://api.github.com/users/filecoin-helper/repos",
    "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
    "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
    "type": "User",
    "site_admin": false
  },
  "node_id": "MDc6UmVsZWFzZTI4MTYyMjA3",
  "tag_name": "6a143e06f923f3a4",
  "target_commitish": "6a143e06f923f3a4f544c7a652e8b4df420a3d28",
  "name": "6a143e06f923f3a4",
  "draft": false,
  "prerelease": false,
  "created_at": "2020-07-02T14:28:44Z",
  "published_at": "2020-07-02T14:35:57Z",
  "assets": [
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437790",
      "id": 22437790,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3Nzkw",
      "name": "filecoin-ffi-Darwin-optimized.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 13138992,
      "download_count": 5,
      "created_at": "2020-07-02T14:41:09Z",
      "updated_at": "2020-07-02T14:41:10Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Darwin-optimized.tar.gz"
    },
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437740",
      "id": 22437740,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3NzQw",
      "name": "filecoin-ffi-Darwin-standard.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 13138995,
      "download_count": 1282,
      "created_at": "2020-07-02T14:40:05Z",
      "updated_at": "2020-07-02T14:40:07Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Darwin-standard.tar.gz"
    },
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437646",
      "id": 22437646,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3NjQ2",
      "name": "filecoin-ffi-Linux-optimized.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 14086829,
      "download_count": 8223,
      "created_at": "2020-07-02T14:36:25Z",
      "updated_at": "2020-07-02T14:36:25Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Linux-optimized.tar.gz"
    },
    {
      "url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437626",
      "id": 22437626,
      "node_id": "MDEyOlJlbGVhc2VBc3NldDIyNDM3NjI2",
      "name": "filecoin-ffi-Linux-standard.tar.gz",
      "label": "",
      "uploader": {
        "login": "filecoin-helper",
        "id": 37553297,
        "node_id": "MDQ6VXNlcjM3NTUzMjk3",
        "avatar_url": "https://avatars.githubusercontent.com/u/37553297?v=4",
        "gravatar_id": "",
        "url": "https://api.github.com/users/filecoin-helper",
        "html_url": "https://github.com/filecoin-helper",
        "followers_url": "https://api.github.com/users/filecoin-helper/followers",
        "following_url": "https://api.github.com/users/filecoin-helper/following{/other_user}",
        "gists_url": "https://api.github.com/users/filecoin-helper/gists{/gist_id}",
        "starred_url": "https://api.github.com/users/filecoin-helper/starred{/owner}{/repo}",
        "subscriptions_url": "https://api.github.com/users/filecoin-helper/subscriptions",
        "organizations_url": "https://api.github.com/users/filecoin-helper/orgs",
        "repos_url": "https://api.github.com/users/filecoin-helper/repos",
        "events_url": "https://api.github.com/users/filecoin-helper/events{/privacy}",
        "received_events_url": "https://api.github.com/users/filecoin-helper/received_events",
        "type": "User",
        "site_admin": false
      },
      "content_type": "application/octet-stream",
      "state": "uploaded",
      "size": 14086828,
      "download_count": 5172,
      "created_at": "2020-07-02T14:35:57Z",
      "updated_at": "2020-07-02T14:35:58Z",
      "browser_download_url": "https://github.com/filecoin-project/filecoin-ffi/releases/download/6a143e06f923f3a4/filecoin-ffi-Linux-standard.tar.gz"
    }
  ],
  "tarball_url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/tarball/6a143e06f923f3a4",
  "zipball_url": "https://api.github.com/repos/filecoin-project/filecoin-ffi/zipball/6a143e06f923f3a4",
  "body": ""
}'
+ local __release_url=https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437646
++ basename https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437646
+ local __tar_path=/tmp/filecoin-ffi-Linux-optimized_22437646.tar.gz
+ [[ -z https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437646 ]]
++ curl --head --retry 3 --header Accept:application/octet-stream --location --output /dev/null -w '%{url_effective}' https://api.github.com/repos/filecoin-project/filecoin-ffi/releases/assets/22437646
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0
  0 13.4M    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0
+ local '__asset_url=https://objects.githubusercontent.com/github-production-release-asset-2e65be/223221398/bb9b6280-bc36-11ea-936f-6dfbbc67c869?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20211218%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20211218T082929Z&X-Amz-Expires=300&X-Amz-Signature=bfe7a97ce2d6f13dfaafad0969774a7c6a25160134d75b0157bd25b7b04f664f&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=223221398&response-content-disposition=attachment%3B%20filename%3Dfilecoin-ffi-Linux-optimized.tar.gz&response-content-type=application%2Foctet-stream'
+ curl --retry 3 --output /tmp/filecoin-ffi-Linux-optimized_22437646.tar.gz 'https://objects.githubusercontent.com/github-production-release-asset-2e65be/223221398/bb9b6280-bc36-11ea-936f-6dfbbc67c869?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20211218%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20211218T082929Z&X-Amz-Expires=300&X-Amz-Signature=bfe7a97ce2d6f13dfaafad0969774a7c6a25160134d75b0157bd25b7b04f664f&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=223221398&response-content-disposition=attachment%3B%20filename%3Dfilecoin-ffi-Linux-optimized.tar.gz&response-content-type=application%2Foctet-stream'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 13.4M  100 13.4M    0     0  72735      0  0:03:13  0:03:13 --:--:-- 82479
+ eval '__tarball_path='\''/tmp/filecoin-ffi-Linux-optimized_22437646.tar.gz'\'''
++ __tarball_path=/tmp/filecoin-ffi-Linux-optimized_22437646.tar.gz
++ mktemp -d
+ local __tmp_dir=/tmp/tmp.kCD60rRotr
+ tar -C /tmp/tmp.kCD60rRotr -xzf /tmp/filecoin-ffi-Linux-optimized_22437646.tar.gz
+ find -L /tmp/tmp.kCD60rRotr -type f -name filcrypto.h -exec cp -- '{}' . ';'
+ find -L /tmp/tmp.kCD60rRotr -type f -name libfilcrypto.a -exec cp -- '{}' . ';'
+ find -L /tmp/tmp.kCD60rRotr -type f -name filcrypto.pc -exec cp -- '{}' . ';'
+ check_installed_files
+ [[ ! -f ./filcrypto.h ]]
+ [[ ! -f ./libfilcrypto.a ]]
+ [[ ! -f ./filcrypto.pc ]]
+ echo '[install-filcrypto/main] successfully installed prebuilt libfilcrypto'
[install-filcrypto/main] successfully installed prebuilt libfilcrypto

```