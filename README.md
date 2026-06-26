# Operating System — Personal Reference Notes

A self-reference tutorial for Operating System concepts written in plain English with real-life analogies, flow diagrams, and structured tables.

---

## Structure

```
01_basics_of_os/
  01_os_explained_purpose_and_objectives.md   # What is an OS? Why we need it, key objectives
  02_process_vs_program_vs_thread.md          # Differences between process, program, and thread
  03_types_of_operating_systems.md            # Batch, Time-sharing, Real-Time, etc.
  04_batch_multiprogramming_multitasking.md   # Batch OS, Multiprogramming, Multitasking explained
  05_multiprocessing_and_rtos.md              # Multiprocessing and Real-Time OS
  06_distributed_clustered_embedded_os.md     # Distributed, Clustered, and Embedded OS

02_process_management/
  01_process_states.md                       # Process concept, 5 states, state transitions
  02_process_control_block.md                # PCB structure and what the OS tracks per process
  03_context_switching.md                    # How the OS saves and restores process state
  04_user_mode_vs_kernel_mode.md             # Privilege levels and mode switching
  05_scheduling_queues_and_schedulers.md     # Ready queue, I/O queue, short/mid/long-term schedulers
  06_preemptive_vs_non_preemptive.md         # Scheduling types and their trade-offs
  07_cpu_scheduling_fcfs_sjf_srtf.md         # FCFS, SJF, SRTF algorithms explained
  08_cpu_scheduling_advanced.md              # Round Robin, Priority, HRRN, MLQ
  09_threads_in_os.md                        # Thread concept, benefits, and types
  10_user_vs_kernel_threads.md               # Multithreading models: many-to-one, one-to-one, many-to-many

03_synchronization_and_concurrency/
  01_process_synchronization.md             # Race conditions, mutual exclusion, synchronization goals
  02_critical_section_problem.md            # Critical section, Peterson's solution
  03_mutex_vs_semaphore.md                  # Mutex and semaphore types with examples
  04_producer_consumer_problem.md           # Classic sync problem solved with semaphores
  05_deadlock_in_os.md                      # Deadlock conditions, prevention, avoidance, detection
  06_bankers_algorithm.md                   # Deadlock avoidance using Banker's algorithm
  07_starvation_and_aging.md               # Starvation causes and aging solution
  08_priority_inversion.md                  # Priority inversion problem and fixes

04_memory_management/
  01_memory_management_and_address_binding.md  # Memory mgmt goals, address types, binding types, MMU, swapping
  02_first_fit_best_fit_worst_fit.md            # Contiguous allocation strategies and fragmentation
  03_paging_vs_segmentation.md                  # Paging, segmentation, and hybrid schemes
  04_virtual_memory_and_demand_paging.md        # Virtual memory concept and demand paging
  05_page_fault_handling.md                     # Page fault lifecycle and OS response
  06_page_replacement_algorithms.md             # FIFO, LRU, Optimal algorithms
  07_beladys_anomaly.md                         # More frames causing more faults
  08_thrashing_and_working_set_model.md         # Thrashing causes and working set fix
  09_dynamic_linking_vs_dynamic_loading.md      # Dynamic linking and loading differences

05_file_systems/
  01_file_system_basics.md                     # File system basics and key components
  02_fat_ntfs_ext4.md                          # FAT, NTFS, and ext4 file system comparison
  03_file_allocation_methods.md                # Contiguous, linked, and indexed allocation
  04_fcb_vs_acl_permissions.md                 # File Control Block vs Access Control List
  05_file_system_security.md                   # File system security and access control
  06_file_system_encryption.md                 # File system encryption techniques
  07_file_system_journaling.md                 # Journaling for crash recovery
  08_backup_and_recovery.md                    # Backup strategies and recovery methods

06_disk_management_and_storage/
  01_disk_scheduling_and_structure.md          # Disk structure and scheduling goals
  02_disk_scheduling_algorithms.md             # FCFS, SSTF, SCAN, C-SCAN, LOOK
  03_raid_levels.md                            # RAID 0 through RAID 6 explained
  04_spooling_and_swap_space.md                # Spooling and swap space management

07_inter_process_communication/
  01_ipc_message_passing_vs_shared_memory.md   # IPC models: message passing vs shared memory
  02_pipes_vs_named_pipes.md                   # Pipes and named pipes
  03_signal_handling.md                        # Signal handling in OS
  04_thread_safety_vs_reentrancy.md            # Thread safety and reentrancy

08_system_internals_and_performance/
  01_system_calls.md                           # System calls: types, uses, examples
  02_booting_process.md                        # Booting process from power-on to OS
  03_hardware_vs_software_interrupts.md        # Hardware vs software interrupts
  04_event_driven_programming.md               # Event-driven programming in OS
  05_resource_management_and_load_balancing.md # Resource management and load balancing
  06_performance_measurement_and_tuning.md     # OS performance measurement and tuning

09_advanced_os_concepts/
  01_cache_mapping.md                          # Cache mapping: direct vs associative
  02_virtual_machines_and_hypervisors.md       # Virtual machines and hypervisors
  03_nfs_network_file_system.md                # NFS: network file system
  04_containers_and_virtualization.md          # Containers and OS-level virtualization
  05_os_security.md                            # Operating system security
  06_future_os_trends.md                       # Future OS trends and innovations

threads_and_parallelism/
  01_concepts_concurrency_and_parallelism.md   # Concurrency vs parallelism concepts
  02_threads_in_cpp.md                         # Thread creation and management in C++
  03_threads_in_python.md                      # Thread creation and management in Python
  04_parallel_processing_vs_threads.md         # Parallel processing vs threading comparison
```
