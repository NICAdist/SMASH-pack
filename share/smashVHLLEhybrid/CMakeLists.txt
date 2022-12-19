# The project name
project(annas-hybrid NONE C)

cmake_minimum_required(VERSION 3.20)

#----------------------------------------------------------------------------#
# Function to set up hybrid evolution depending on system and energy
#----------------------------------------------------------------------------#

if (SMASH_SETTINGS) # if user provided file with settings, read them
  get_filename_component(settingsFullPath "${SMASH_SETTINGS}" REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")
  include(${settingsFullPath})
endif()

if (${VAL_NUM_RUNS_ONE_ENERGY})
  set(NUM_RUNS_ONE_ENERGY ${VAL_NUM_RUNS_ONE_ENERGY})
else()
  set(NUM_RUNS_ONE_ENERGY "100")
endif()

if (${VAL_NUM_RUNS_CUSTOM_HYBRID})
  set(NUM_RUNS_CUSTOM_HYBRID ${VAL_NUM_RUNS_CUSTOM_HYBRID})
else()
  set(NUM_RUNS_CUSTOM_HYBRID "100")
endif()

if (${VAL_NUM_RUNS_TEST_HYBRID})
  set(NUM_RUNS_TEST_HYBRID ${VAL_NUM_RUNS_TEST_HYBRID})
else()
  set(NUM_RUNS_TEST_HYBRID "4")
endif()

if (${VAL_NUM_EVENTS_AFTERBURNER_ENERGY})
  set(NUM_EVENTS_AFTERBURNER_ENERGY ${VAL_NUM_EVENTS_AFTERBURNER_ENERGY})
else()
  set(NUM_EVENTS_AFTERBURNER_ENERGY "1000")
endif()

if (${VAL_NUM_EVENTS_AFTERBURNER_CUSTOM})
  set(NUM_EVENTS_AFTERBURNER_CUSTOM ${VAL_NUM_EVENTS_AFTERBURNER_CUSTOM})
else()
  set(NUM_EVENTS_AFTERBURNER_CUSTOM "1000")
endif()

if (${VAL_NUM_EVENTS_AFTERBURNER_TEST})
  set(NUM_EVENTS_AFTERBURNER_TEST ${VAL_NUM_EVENTS_AFTERBURNER_TEST})
else()
  set(NUM_EVENTS_AFTERBURNER_TEST "50")
endif()

function(run_one_energy
        energy_and_system
        )
  list(GET energy_and_system 0 energy)
  list(GET energy_and_system 1 system)
  message(STATUS "Configuring hybrid run for ${system} @ sqrt(s) = ${energy} GeV.")
  set(results_folder "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results/${system}_${energy}")
  file(MAKE_DIRECTORY ${results_folder})

  set(smash_IC_files "")
  set(vhlle_freezeout_hypersurfaces "")
  set(sampled_particle_lists "")
  set(final_particle_lists "")
  set(all_analysis_outputs "")
  set(all_plots "")
  set(all_conservation_plots "")

  foreach(i RANGE 1 ${NUM_RUNS_ONE_ENERGY})
    file(MAKE_DIRECTORY "${results_folder}/${i}/IC")       # subdir for initial conditions
    file(MAKE_DIRECTORY "${results_folder}/${i}/Hydro")    # subdir for hydro run
    file(MAKE_DIRECTORY "${results_folder}/${i}/Sampler")  # subdir for particle sampling
    file(MAKE_DIRECTORY "${results_folder}/${i}/Afterburner")  # subdir for afterburner
    file(MAKE_DIRECTORY "${results_folder}/${i}/Spectra")  # subdir for analyzed spectra and plots
    # subdir for tracking simulation progress and description of it
    file(MAKE_DIRECTORY "${results_folder}/${i}/Progress")
    FILE(WRITE ${results_folder}/${i}/Progress/readme.txt
    "This folder contains empty files created after finishing each step of simulation toolchain.\n"
    "Removing them will force to rebuild a given part of toolchain. No results of simulation are\n"
    "stored here.")
    # Set variables for files
    set(smash_IC_file "${results_folder}/${i}/IC/SMASH_IC.dat")
    set(smash_IC_oscar "${results_folder}/${i}/IC/SMASH_IC.oscar")
    set(smash_IC_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/smash_initial_conditions_${system}.yaml")
    set(vhlle_default_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/vhlle_hydro")
    set(vhlle_output_dir "${results_folder}/${i}/Hydro/")
    set(vhlle_updated_config "${results_folder}/${i}/Hydro/vhlle_config")
    set(vhlle_freezeout_hypersurface "${results_folder}/${i}/Hydro/freezeout.dat")
    set(sampler_input_hypersurface "${results_folder}/${i}/Sampler/freezeout.dat")
    set(sampler_default_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/hadron_sampler")
    set(sampler_updated_config "${results_folder}/${i}/Sampler/sampler_config")
    set(sampler_dir "${results_folder}/${i}/Sampler")
    set(sampled_particle_list "${results_folder}/${i}/Sampler/particle_lists.oscar")
    set(full_particle_list "${results_folder}/${i}/Sampler/sampling0")
    set(smash_afterburner_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/smash_afterburner.yaml")
    set(final_particle_list "${results_folder}/${i}/Afterburner/particles_binary.bin")

    list(APPEND smash_IC_files ${smash_IC_file})
    list(APPEND vhlle_freezeout_hypersurfaces ${vhlle_freezeout_hypersurface})
    list(APPEND sampled_particle_lists ${sampled_particle_list})
    list(APPEND final_particle_lists ${final_particle_list})

    #----------------------------------------------------------------------------#
    # Run SMASH and generate initial conditions output
    #----------------------------------------------------------------------------#
    add_custom_command(
         OUTPUT "${smash_IC_file}"
                "${results_folder}/${i}/config.yaml"
                "${results_folder}/${i}/Progress/f1.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/smash"
                "-o" "${results_folder}/${i}/IC"
                "-i" "${smash_IC_config}"
                "-c" "Modi: {Collider: {Sqrtsnn: ${energy}}}"
                "-f"
                "-n"
                ">" "${results_folder}/${i}/IC/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f1.txt"
        DEPENDS "${smash_IC_config}"
                "${SMASH_PACK_ROOT}/bin/smash"
        COMMENT "Running SMASH for Initial Conditions of ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f1" ALL DEPENDS "${results_folder}/${i}/Progress/f1.txt")

    #----------------------------------------------------------------------------#
    # Feed SMASH initial conditions into vHLLE and run hydro evolution
    #----------------------------------------------------------------------------#
    # create input file with correct paths for vHLLE
    add_custom_command(
         OUTPUT "${vhlle_updated_config}"
                "${results_folder}/${i}/Progress/f2.txt"
         COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_vhlle_config.py"
                "--vhlle_config" "${vhlle_default_config}"
                "--smash_ic" "${smash_IC_file}"
                "--output_file" "${vhlle_updated_config}"
                "--energy" "${energy}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f2.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_vhlle_config.py"
                "${vhlle_default_config}"
                "${results_folder}/${i}/Progress/f1.txt"
        COMMENT "Creating input file for vHLLE for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f2" ALL DEPENDS "${results_folder}/${i}/Progress/f2.txt")
    add_dependencies("${system}_${energy}_${i}_f2" "${system}_${energy}_${i}_f1")
    
    # Run hydro
    add_custom_command(
         OUTPUT "${vhlle_freezeout_hypersurface}"
                "${results_folder}/${i}/Progress/f3.txt"
        COMMAND "${VHLLE_ROOT}/bin/hlle_visc"
                "-params" "${vhlle_updated_config}"
                "-ISinput" "${smash_IC_file}"
                "-outputDir" "${vhlle_output_dir}"
                "-configDir" "${VHLLE_PARAMS}"
                ">" "${vhlle_output_dir}/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f3.txt"
        DEPENDS "${vhlle_updated_config}"
                "${VHLLE_ROOT}/bin/hlle_visc"
                "${results_folder}/${i}/Progress/f2.txt"
        COMMENT "Running vHLLE for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )   
    add_custom_target("${system}_${energy}_${i}_f3" ALL DEPENDS "${results_folder}/${i}/Progress/f3.txt")
    add_dependencies("${system}_${energy}_${i}_f3" "${system}_${energy}_${i}_f2")

    #----------------------------------------------------------------------------#
    # Run Cooper-Frye sampler for particlization of freezeout surface
    #----------------------------------------------------------------------------#
    # create input file with correct paths for sampler
    add_custom_command(
         OUTPUT "${sampler_updated_config}"
                "${results_folder}/${i}/Progress/f4.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_sampler_config.py"
                "--sampler_config" "${sampler_default_config}"
                "--vhlle_config" "${vhlle_updated_config}"
                "--output_file" "${sampler_updated_config}"
                "--Nevents" "${NUM_EVENTS_AFTERBURNER_ENERGY}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f4.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_sampler_config.py"
                "${sampler_default_config}"
                "${results_folder}/${i}/Progress/f3.txt"
        COMMENT "Creating input file for sampler for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f4" ALL DEPENDS "${results_folder}/${i}/Progress/f4.txt")
    add_dependencies("${system}_${energy}_${i}_f4" "${system}_${energy}_${i}_f3")

    # Run sampler
    add_custom_command(
         OUTPUT ${sampled_particle_list}
                "${results_folder}/${i}/Progress/f5.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/hadron_sampler" "events" "1" "${sampler_updated_config}"
                ">" "${results_folder}/${i}/Sampler/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f5.txt"
        DEPENDS "${SMASH_PACK_ROOT}/bin/hadron_sampler"
                "${sampler_updated_config}"
                "${results_folder}/${i}/Progress/f4.txt"
        COMMENT "Sampling particles from freezeout hypersurface (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f5" ALL DEPENDS "${results_folder}/${i}/Progress/f5.txt")
    add_dependencies("${system}_${energy}_${i}_f5" "${system}_${energy}_${i}_f4")

    #----------------------------------------------------------------------------#
    # Run SMASH as afterburner
    #----------------------------------------------------------------------------#
    # Add spectators to particle list and rename it to be in accordance
    # with SMASH list modus input format
    add_custom_command(
         OUTPUT ${full_particle_list}
                "${results_folder}/${i}/Progress/f6.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/add_spectators.py"
                "--sampled_particle_list" "${sampler_dir}/particle_lists.oscar"
                "--initial_particle_list" "${smash_IC_oscar}"
                "--output_file" "${full_particle_list}"
                "--smash_config" "${results_folder}/${i}/IC/config.yaml"
        COMMAND "touch" "${results_folder}/${i}/Progress/f6.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/add_spectators.py"
                "${results_folder}/${i}/Progress/f5.txt"
        COMMENT "Adding spectators to sampled particle list for afterburner (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f6" ALL DEPENDS "${results_folder}/${i}/Progress/f6.txt")
    add_dependencies("${system}_${energy}_${i}_f6" "${system}_${energy}_${i}_f5")

    # Run afterburner evolution
    add_custom_command(
         OUTPUT "${final_particle_list}"
                "${results_folder}/${i}/Afterburner/config.yaml"
                "${results_folder}/${i}/Progress/f7.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/smash"
                "-i" "${smash_afterburner_config}"
                "-c" "Modi: { List: { File_Directory: ${sampler_dir}} }"
                "-c" "General: { Nevents: ${NUM_EVENTS_AFTERBURNER_ENERGY} }"
                "-o" "${results_folder}/${i}/Afterburner"
                "-f"
                "-n"
                ">" "/dev/null"
        COMMAND "touch" "${results_folder}/${i}/Progress/f7.txt"
        DEPENDS "${smash_afterburner_config}"
                "${SMASH_PACK_ROOT}/bin/smash"
                "${full_particle_list}"
                "${results_folder}/${i}/Progress/f6.txt"
        COMMENT "Running SMASH afterburner for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
    )
    add_custom_target("${system}_${energy}_${i}_f7" ALL DEPENDS "${results_folder}/${i}/Progress/f7.txt")
    add_dependencies("${system}_${energy}_${i}_f7" "${system}_${energy}_${i}_f6")

    #----------------------------------------------------------------------------#
    # Analyse data
    #----------------------------------------------------------------------------#
    # Define filenames for particle spectra and create list of those
    set(spectra_fnames "${results_folder}/${i}/Spectra/yspectra.txt" ;
                       "${results_folder}/${i}/Spectra/mtspectra.txt" ;
                       "${results_folder}/${i}/Spectra/ptspectra.txt" ;
                       "${results_folder}/${i}/Spectra/v2spectra.txt" ;
                       "${results_folder}/${i}/Spectra/meanmt0_midrapidity.txt" ;
                       "${results_folder}/${i}/Spectra/meanpt_midrapidity.txt" ;
                       "${results_folder}/${i}/Spectra/midrapidity_yield.txt" ;
                       "${results_folder}/${i}/Spectra/total_multiplicity.txt")
    set(analysis_outputs "")
    foreach(j ${spectra_fnames})
        list(APPEND analysis_outputs "${j}")
    endforeach(j)

    list(APPEND all_analysis_outputs ${analysis_outputs})

    # Perform analysis
    add_custom_command(
         OUTPUT ${analysis_outputs}
                "${results_folder}/${i}/Progress/f8.txt"
        COMMAND "python3" "${SMASH_PACK_ROOT}/share/smashAnalysis/test/energy_scan/mult_and_spectra.py"
                "--output_files" ${analysis_outputs}
                "--input_files" "${final_particle_list}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f8.txt"
        DEPENDS "${SMASH_PACK_ROOT}/share/smashAnalysis/test/energy_scan/mult_and_spectra.py"
                "${results_folder}/${i}/Progress/f7.txt"
        COMMENT "Analyzing spectra for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f8" ALL DEPENDS "${results_folder}/${i}/Progress/f8.txt")
    add_dependencies("${system}_${energy}_${i}_f8" "${system}_${energy}_${i}_f7")
    
    #----------------------------------------------------------------------------#
    # Plot spectra
    #----------------------------------------------------------------------------#
    # Define plot names
    set(plot_names "${results_folder}/${i}/Spectra/yspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/mtspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/ptspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/v2spectra.pdf" ;
                   "${results_folder}/${i}/Spectra/meanmt0_midrapidity.pdf" ;
                   "${results_folder}/${i}/Spectra/meanpt_midrapidity.pdf" ;
                   "${results_folder}/${i}/Spectra/midrapidity_yield.pdf" ;
                   "${results_folder}/${i}/Spectra/total_multiplicity.pdf")

    set(plots "")
    foreach(j ${plot_names})
       list(APPEND plots "${j}")
    endforeach(j)

    list(APPEND all_plots ${plots})

    # Perform plotting
    add_custom_command(
         OUTPUT ${plots}
                "${results_folder}/${i}/Progress/f9.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_spectra.py"
                "--input_files" "${results_folder}/${i}/Spectra/*.txt"
                "--energy" "${energy}"
                "--system" "${system}"
                "--Nevents" "${NUM_EVENTS_AFTERBURNER_ENERGY}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f9.txt"
        DEPENDS "${results_folder}/${i}/Progress/f8.txt"
        COMMENT "Plotting spectra for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f9" ALL DEPENDS "${results_folder}/${i}/Progress/f9.txt")
    add_dependencies("${system}_${energy}_${i}_f9" "${system}_${energy}_${i}_f8")
    
    #----------------------------------------------------------------------------#
    # Check conserved quantities
    #----------------------------------------------------------------------------#
    set(smash_IC_binary "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results/${system}_${energy}/${i}/IC/SMASH_IC.bin")
    set(output_dir "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results/${system}_${energy}/${i}/")
    set(E_cons_plot "${output_dir}/Energy_Conservation.pdf")
    set(Mult_plot "${output_dir}/Pions.pdf")
    set(Hydro_lifetime_list "${output_dir}/Hydro_lifetime.txt")

    list(APPEND all_conservation_plots ${E_cons_plot})
    list(APPEND all_multiplicity_plots ${Mult_plot})
    list(APPEND all_lifetime_lists ${Hydro_lifetime_list})
    add_custom_command(
         OUTPUT ${E_cons_plot}
                "${results_folder}/${i}/Progress/f10.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/check_conservation.py"
                "--SMASH_IC" "${smash_IC_oscar}"
                "--Hydro_Info" "${results_folder}/${i}/Hydro/Terminal_Output.txt"
                "--Sampler" "${full_particle_list}"
                "--SMASH_final_state" "${final_particle_list}"
                "--SMASH_ana_path" "${SMASH_PACK_ROOT}/share/smashAnalysis"
                "--output_path" "${output_dir}"
                "--Nevents" "${NUM_EVENTS_AFTERBURNER_ENERGY}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f10.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/check_conservation.py"
                "${results_folder}/${i}/Progress/f9.txt"
        COMMENT "Checking conservation laws for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f10" ALL DEPENDS "${results_folder}/${i}/Progress/f10.txt")
    add_dependencies("${system}_${energy}_${i}_f10" "${system}_${energy}_${i}_f9")
    
    add_custom_command(
         OUTPUT ${Mult_plot}
                "${results_folder}/${i}/Progress/f11.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/check_multiplicities.py"
                "--Freezeout_Surface" "${vhlle_freezeout_hypersurface}"
                "--Sampler" "${sampled_particle_list}"
                "--output_path" "${output_dir}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f11.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/check_multiplicities.py"
                "${results_folder}/${i}/Progress/f10.txt"
        COMMENT "Verifying multiplicities for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f11" ALL DEPENDS "${results_folder}/${i}/Progress/f11.txt")
    add_dependencies("${system}_${energy}_${i}_f11" "${system}_${energy}_${i}_f10")

    add_custom_command(
         OUTPUT ${Hydro_lifetime_list}
                "${results_folder}/${i}/Progress/f12.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/get_hydro_endtime.py"
                "--Hydro_Info" "${results_folder}/${i}/Hydro/Terminal_Output.txt"
                "--output_path" "${output_dir}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f12.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/get_hydro_endtime.py"
                "${results_folder}/${i}/Progress/f11.txt"
        COMMENT "Find hydro lifetime in ${system} @ ${energy} GeV (${i}/${NUM_RUNS_ONE_ENERGY})."
        )
    add_custom_target("${system}_${energy}_${i}_f12" ALL DEPENDS "${results_folder}/${i}/Progress/f12.txt")
    add_dependencies("${system}_${energy}_${i}_f12" "${system}_${energy}_${i}_f11")
    endforeach(i)

  #----------------------------------------------------------------------------#
  # Average previously obtained spectra
  #----------------------------------------------------------------------------#
  # subdir for tracking simulation progress and description of it
  file(MAKE_DIRECTORY "${results_folder}/Progress")
  FILE(WRITE ${results_folder}/Progress/readme.txt
  "This folder contains empty files created after finishing each step of simulation toolchain.\n"
  "Removing them will force to rebuild a given part of toolchain. No results of simulation are\n"
  "stored here.")
  file(MAKE_DIRECTORY "${results_folder}/Averaged_Spectra")
  set(averaged_spectra "${results_folder}/pT.txt" ;
                       "${results_folder}/mT.txt" ;
                       "${results_folder}/dNdy.txt"
                     )
  add_custom_command(
       OUTPUT ${averaged_spectra}
              "${results_folder}/Progress/f13.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/average_spectra.py"
              "--smash_ana_dir" "${SMASH_PACK_ROOT}/share/smashAnalysis"
              "--pT_files" "${results_folder}/*/Spectra/ptspectra.txt"
              "--mT_files" "${results_folder}/*/Spectra/mtspectra.txt"
              "--y_files" "${results_folder}/*/Spectra/yspectra.txt"
              "--midyY_files" "${results_folder}/*/Spectra/midrapidity_yield.txt"
              "--meanpT_files" "${results_folder}/*/Spectra/meanpt_midrapidity.txt"
              "--v2_files" "${results_folder}/*/Spectra/v2spectra.txt"
              "--Nevents" "${NUM_EVENTS_AFTERBURNER_ENERGY}"
              "--output_dir" "${results_folder}/Averaged_Spectra/"
      COMMAND "touch" "${results_folder}/Progress/f13.txt"
      COMMENT "Averaging spectra for ${system} @ ${energy} GeV."
      )
  add_custom_target("${system}_${energy}_f13" ALL DEPENDS "${results_folder}/Progress/f13.txt")

  #----------------------------------------------------------------------------#
  # Average previously obtained flow
  #----------------------------------------------------------------------------#
  set(averaged_flow_spectra "${results_folder}/v2.txt" ;
                            "${results_folder}/v3.txt" ;
                     )
  add_custom_command(
       OUTPUT ${averaged_flow_spectra}
              "${results_folder}/Progress/f14.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/average_flow.py"
              "--v2_files" "${results_folder}/*/Spectra/EP_v2.txt"
              "--v3_files" "${results_folder}/*/Spectra/EP_v3.txt"
              "--energy" "${energy}"
              "--output_dir" "${results_folder}/Averaged_Spectra/"
      COMMAND "touch" "${results_folder}/Progress/f14.txt"
      DEPENDS "${results_folder}/Progress/f13.txt"
      COMMENT "Averaging flow for ${system} @ ${energy} GeV."
      )
  add_custom_target("${system}_${energy}_f14" ALL DEPENDS "${results_folder}/Progress/f14.txt")
  add_dependencies("${system}_${energy}_f14" "${system}_${energy}_f13")

  #----------------------------------------------------------------------------#
  # Average previously obtained endtimes
  #----------------------------------------------------------------------------#
  set(averaged_endtime_files "${results_folder}/Endtime.txt")
  add_custom_command(
       OUTPUT ${averaged_endtime_files}
              "${results_folder}/Progress/f15.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/average_endtimes.py"
              "--endtime_files" "${results_folder}/*/Hydro_Endtime.txt"
              "--energy" "${energy}"
              "--output_dir" "${results_folder}/Averaged_Spectra/"
      COMMAND "touch" "${results_folder}/Progress/f15.txt"
      DEPENDS "${results_folder}/Progress/f14.txt"
      COMMENT "Averaging flow for ${system} @ ${energy} GeV."
      )
  add_custom_target("${system}_${energy}_f15" ALL DEPENDS "${results_folder}/Progress/f15.txt")
  add_dependencies("${system}_${energy}_f15" "${system}_${energy}_f14")

  #----------------------------------------------------------------------------#
  # Plot averaged quantities
  #----------------------------------------------------------------------------#
  set(averaged_plots "${results_folder}/Averaged_Spectra/yspectra.pdf" ;
                     "${results_folder}/Averaged_Spectra/mtspectra.pdf" ;
                     "${results_folder}/Averaged_Spectra/ptspectra.pdf")

  # Perform plotting of averaged quantities
  add_custom_command(
       OUTPUT ${averaged_plots}
              "${results_folder}/Progress/f16.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_spectra.py"
              "--input_files" "${results_folder}/Averaged_Spectra/*.txt"
              "--energy" "${energy}"
              "--system" "${system}"
              "--Nevents" "${NUM_EVENTS_AFTERBURNER_ENERGY}"
              "--average" "True"
      COMMAND "touch" "${results_folder}/Progress/f16.txt"
      DEPENDS "${results_folder}/Progress/f15.txt"
      COMMENT "Plotting averaged spectra for ${system} @ ${energy} GeV."
      )
  add_custom_target("${system}_${energy}_f16" ALL DEPENDS "${results_folder}/Progress/f16.txt")
  add_dependencies("${system}_${energy}_f16" "${system}_${energy}_f15")

  # Define subtargets to enable separated running of the hybrid submodules
  set(target ${system}_${energy}_IC)
  add_custom_target(${target} ALL DEPENDS ${smash_IC_files})

  set(target ${system}_${energy}_hydro)
  add_custom_target(${target} ALL DEPENDS ${vhlle_freezeout_hypersurfaces})

  set(target ${system}_${energy}_sampler)
  add_custom_target(${target} ALL DEPENDS ${sampled_particle_lists})

  set(target ${system}_${energy}_afterburner)
  add_custom_target(${target} ALL DEPENDS ${final_particle_lists})

  set(target ${system}_${energy}_analysis)
  add_custom_target(${target} ALL DEPENDS ${all_analysis_outputs})

  set(target ${system}_${energy}_plots)
  add_custom_target(${target} ALL DEPENDS ${all_plots})

  set(target ${system}_${energy}_check_conservation)
  add_custom_target(${target} ALL DEPENDS ${all_conservation_plots})

  set(target ${system}_${energy}_check_multiplicities)
  add_custom_target(${target} ALL DEPENDS ${all_multiplicity_plots})

  set(target ${system}_${energy}_hydro_endtime)
  add_custom_target(${target} ALL DEPENDS ${all_lifetime_lists})

  # Averaging
  set(target ${system}_${energy}_average_spectra)
  add_custom_target(${target} ALL DEPENDS ${averaged_spectra})

  set(target ${system}_${energy}_average_flow)
  add_custom_target(${target} ALL DEPENDS ${averaged_flow_spectra})

  set(target ${system}_${energy}_average_plots)
  add_custom_target(${target} ALL DEPENDS ${averaged_plots})

  set(target ${system}_${energy}_average_endtime)
  add_custom_target(${target} ALL DEPENDS ${averaged_endtime_files})
endfunction()

function(Excitation_Funcs
        )
  set(outputdir "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results")
  set(midY_plot "${outputdir}/midy_yield_exc_func.pdf")
  add_custom_command(
       OUTPUT ${midY_plot}
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_excitation_functions.py"
              "--midyY_files" "${outputdir}/*/Averaged_Spectra/midyY.txt"
              "--v2_files" "${outputdir}/*/Averaged_Spectra/v2spectra.txt"
              "--meanpT_files" "${outputdir}/*/Averaged_Spectra/meanpT.txt"
              "--output_dir" "${outputdir}/"
      DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_excitation_functions.py"
      COMMENT "Plotting excitation functions."
      )
  set(target exc_funcs)
  add_custom_target(${target} ALL DEPENDS ${midY_plot})
endfunction()

function(Integrated_vn
        )
  set(outputdir "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results")
  set(int_vn_plot "${outputdir}/int_vn.pdf")
  add_custom_command(
       OUTPUT ${int_vn_plot}
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_int_vn.py"
              "--vn_files" "${outputdir}/*/Averaged_Spectra/int_vn.txt"
              "--output_dir" "${outputdir}/"
      DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_int_vn.py"
      COMMENT "Plotting integrated vn excitation functions."
      )
  set(target int_vn)
  add_custom_target(${target} ALL DEPENDS ${int_vn_plot})
endfunction()

function(Hydro_Lifetimes
        )
  set(outputdir "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results")
  set(lifetime_file "${outputdir}/hydro_lifetime.txt")
  add_custom_command(
       OUTPUT ${lifetime_file}
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/assemble_endtimes.py"
              "--endtime_files" "${outputdir}/*/Averaged_Spectra/Endtime.txt"
              "--output_dir" "${outputdir}/"
      DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/assemble_endtimes.py"
      COMMENT "Plotting integrated vn excitation functions."
      )
  set(target hydro_lifetimes)
  add_custom_target(${target} ALL DEPENDS ${lifetime_file})
endfunction()

################################################################################
#
# Add functionality to use any custom SMASH config for the initial conditions
#
################################################################################
function(run_custom_hybrid
        )
  message(STATUS "Configuring hybrid run for custom IC config.")
  set(results_folder "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results/Custom")
  file(MAKE_DIRECTORY ${results_folder})

  set(smash_IC_files "")
  set(vhlle_freezeout_hypersurfaces "")
  set(sampled_particle_lists "")
  set(final_particle_lists "")
  set(all_analysis_outputs "")
  set(all_plots "")

  foreach(i RANGE 1 ${NUM_RUNS_CUSTOM_HYBRID})
    file(MAKE_DIRECTORY "${results_folder}/${i}/IC")       # subdir for initial conditions
    file(MAKE_DIRECTORY "${results_folder}/${i}/Hydro")    # subdir for hydro run
    file(MAKE_DIRECTORY "${results_folder}/${i}/Sampler")  # subdir for particle sampling
    file(MAKE_DIRECTORY "${results_folder}/${i}/Afterburner")  # subdir for afterburner
    file(MAKE_DIRECTORY "${results_folder}/${i}/Spectra")  # subdir for analyzed spectra and plots
    # subdir for tracking simulation progress and description of it
    file(MAKE_DIRECTORY "${results_folder}/${i}/Progress")
    FILE(WRITE ${results_folder}/${i}/Progress/readme.txt
    "This folder contains empty files created after finishing each step of simulation toolchain.\n"
    "Removing them will force to rebuild a given part of toolchain. No results of simulation are\n"
    "stored here.")
    set(smash_IC_file "${results_folder}/${i}/IC/SMASH_IC.dat")
    set(smash_IC_oscar "${results_folder}/${i}/IC/SMASH_IC.oscar")
    set(smash_IC_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/smash_initial_conditions_custom.yaml")
    set(vhlle_default_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/vhlle_hydro")
    set(vhlle_output_dir "${results_folder}/${i}/Hydro/")
    set(vhlle_updated_config "${results_folder}/${i}/Hydro/vhlle_config")
    set(vhlle_freezeout_hypersurface "${results_folder}/${i}/Hydro/freezeout.dat")
    set(sampler_input_hypersurface "${results_folder}/${i}/Sampler/freezeout.dat")
    set(sampler_default_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/hadron_sampler")
    set(sampler_updated_config "${results_folder}/${i}/Sampler/sampler_config")
    set(sampler_dir "${results_folder}/${i}/Sampler")
    set(sampled_particle_list "${results_folder}/${i}/Sampler/particle_lists.oscar")
    set(full_particle_list "${results_folder}/${i}/Sampler/sampling0")
    set(smash_afterburner_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/smash_afterburner.yaml")
    set(final_particle_list "${results_folder}/${i}/Afterburner/particles_binary.bin")

    list(APPEND smash_IC_files ${smash_IC_file})
    list(APPEND vhlle_freezeout_hypersurfaces ${vhlle_freezeout_hypersurface})
    list(APPEND sampled_particle_lists ${sampled_particle_list})
    list(APPEND final_particle_lists ${final_particle_list})

    #----------------------------------------------------------------------------#
    # Run SMASH and generate initial conditions output
    #----------------------------------------------------------------------------#
    add_custom_command(
         OUTPUT "${smash_IC_file}"
                "${results_folder}/${i}/IC/config.yaml"
                "${results_folder}/${i}/Progress/f1.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/smash"
                "-o" "${results_folder}/${i}/IC"
                "-i" "${smash_IC_config}"
                "-f"
                "-n"
                ">" "${results_folder}/${i}/IC/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f1.txt"
        DEPENDS "${smash_IC_config}"
                "${SMASH_PACK_ROOT}/bin/smash"
        COMMENT "Running SMASH for initial conditions of custom hybrid run."
        )
    add_custom_target("custom_${i}_f1" ALL DEPENDS "${results_folder}/${i}/Progress/f1.txt")

    #----------------------------------------------------------------------------#
    # Feed SMASH initial conditions into vHLLE and run hydro evolution
    #----------------------------------------------------------------------------#
    # create input file with correct paths for vHLLE
    add_custom_command(
         OUTPUT "${vhlle_updated_config}"
                "${results_folder}/${i}/Progress/f2.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_vhlle_config.py"
                "--vhlle_config" "${vhlle_default_config}"
                "--smash_ic" "${smash_IC_file}"
                "--output_file" "${vhlle_updated_config}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f2.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_vhlle_config.py"
                "${vhlle_default_config}"
                "${results_folder}/${i}/Progress/f1.txt"
        COMMENT "Creating input file for vHLLE for custom hybrid run."
        )
    add_custom_target("custom_${i}_f2" ALL DEPENDS "${results_folder}/${i}/Progress/f2.txt")
    add_dependencies("custom_${i}_f2" "custom_${i}_f1")

    # Run hydro
    add_custom_command(
         OUTPUT "${vhlle_freezeout_hypersurface}"
                "${results_folder}/${i}/Progress/f3.txt"
        COMMAND "${VHLLE_ROOT}/bin/hlle_visc"
                "-params" "${vhlle_updated_config}"
                "-ISinput" "${smash_IC_file}"
                "-outputDir" "${vhlle_output_dir}"
                "-configDir" "${VHLLE_PARAMS}"
                ">" "${vhlle_output_dir}/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f3.txt"
        DEPENDS "${vhlle_updated_config}"
                "${VHLLE_ROOT}/bin/hlle_visc"
                "${results_folder}/${i}/Progress/f2.txt"
        COMMENT "Running vHLLE for custom hybrid run."
        )
    add_custom_target("custom_${i}_f3" ALL DEPENDS "${results_folder}/${i}/Progress/f3.txt")
    add_dependencies("custom_${i}_f3" "custom_${i}_f2")

    #----------------------------------------------------------------------------#
    # Run Cooper-Frye sampler for particlization of freezeout surface
    #----------------------------------------------------------------------------#
    # create input file with correct paths for sampler
    add_custom_command(
         OUTPUT "${sampler_updated_config}"
                "${results_folder}/${i}/Progress/f4.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_sampler_config.py"
                "--sampler_config" "${sampler_default_config}"
                "--vhlle_config" "${vhlle_updated_config}"
                "--output_file" "${sampler_updated_config}"
                "--Nevents" "${NUM_EVENTS_AFTERBURNER_CUSTOM}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f4.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_sampler_config.py"
                "${sampler_default_config}"
                "${results_folder}/${i}/Progress/f3.txt"
        COMMENT "Creating input file for custom hybrid run."
        )
    add_custom_target("custom_${i}_f4" ALL DEPENDS "${results_folder}/${i}/Progress/f4.txt")
    add_dependencies("custom_${i}_f4" "custom_${i}_f3")

    # Run sampler
    add_custom_command(
         OUTPUT ${sampled_particle_list}
                "${results_folder}/${i}/Progress/f5.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/hadron_sampler" "events" "1" "${sampler_updated_config}"
                ">" "${results_folder}/${i}/Sampler/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f5.txt"
        DEPENDS "${SMASH_PACK_ROOT}/bin/hadron_sampler"
                "${sampler_updated_config}"
                "${results_folder}/${i}/Progress/f4.txt"
        COMMENT "Sampling particles from freezeout hypersurface (${i}/${NUM_RUNS_CUSTOM_HYBRID})."
        )
    add_custom_target("custom_${i}_f5" ALL DEPENDS "${results_folder}/${i}/Progress/f5.txt")
    add_dependencies("custom_${i}_f5" "custom_${i}_f4")

    #----------------------------------------------------------------------------#
    # Run SMASH as afterburner
    #----------------------------------------------------------------------------#
    # Add spectators to particle list and rename it to be in accordance
    # with SMASH list modus input format
    add_custom_command(
         OUTPUT ${full_particle_list}
                "${results_folder}/${i}/Progress/f6.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/add_spectators.py"
                "--sampled_particle_list" "${sampler_dir}/particle_lists.oscar"
                "--initial_particle_list" "${smash_IC_oscar}"
                "--output_file" "${full_particle_list}"
                "--smash_config" "${results_folder}/${i}/IC/config.yaml"
        COMMAND "touch" "${results_folder}/${i}/Progress/f6.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/add_spectators.py"
                "${results_folder}/${i}/Progress/f5.txt"
        COMMENT "Adding spectators to sampled particle list for afterburner (${i}/${NUM_RUNS_CUSTOM_HYBRID})."
        )
    add_custom_target("custom_${i}_f6" ALL DEPENDS "${results_folder}/${i}/Progress/f6.txt")
    add_dependencies("custom_${i}_f6" "custom_${i}_f5")

    # Run afterburner evolution
    add_custom_command(
         OUTPUT "${final_particle_list}"
                "${results_folder}/${i}/Afterburner/config.yaml"
                "${results_folder}/${i}/Progress/f7.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/smash"
                "-i" "${smash_afterburner_config}"
                "-c" "Modi: { List: { File_Directory: ${sampler_dir}} }"
                "-c" "General: { Nevents: ${NUM_EVENTS_AFTERBURNER_CUSTOM} }"
                "-o" "${results_folder}/${i}/Afterburner"
                "-f"
                "-n"
                ">" "/dev/null"
        COMMAND "touch" "${results_folder}/${i}/Progress/f7.txt"
        DEPENDS "${smash_afterburner_config}"
                "${SMASH_PACK_ROOT}/bin/smash"
                "${full_particle_list}"
                "${results_folder}/${i}/Progress/f6.txt"
        COMMENT "Running SMASH afterburner for custom hybrid run."
        )
    add_custom_target("custom_${i}_f7" ALL DEPENDS "${results_folder}/${i}/Progress/f7.txt")
    add_dependencies("custom_${i}_f7" "custom_${i}_f6")

    #----------------------------------------------------------------------------#
    # Analyse data
    #----------------------------------------------------------------------------#
    # Define filenames for particle spectra and create list of those
    set(spectra_fnames "${results_folder}/${i}/Spectra/yspectra.txt" ;
                       "${results_folder}/${i}/Spectra/mtspectra.txt" ;
                       "${results_folder}/${i}/Spectra/ptspectra.txt" ;
                       "${results_folder}/${i}/Spectra/v2spectra.txt" ;
                       "${results_folder}/${i}/Spectra/meanmt0_midrapidity.txt" ;
                       "${results_folder}/${i}/Spectra/meanpt_midrapidity.txt" ;
                       "${results_folder}/${i}/Spectra/midrapidity_yield.txt" ;
                       "${results_folder}/${i}/Spectra/total_multiplicity.txt")
    set(analysis_outputs "")
    foreach(j ${spectra_fnames})
        list(APPEND analysis_outputs "${j}")
    endforeach(j)

    list(APPEND all_analysis_outputs ${analysis_outputs})

    # Perform analysis
    add_custom_command(
         OUTPUT ${analysis_outputs}
                "${results_folder}/${i}/Progress/f8.txt"
        COMMAND "python3" "${SMASH_PACK_ROOT}/share/smashAnalysis/test/energy_scan/mult_and_spectra.py"
                "--output_files" ${analysis_outputs}
                "--input_files" "${final_particle_list}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f8.txt"
        DEPENDS "${SMASH_PACK_ROOT}/share/smashAnalysis/test/energy_scan/mult_and_spectra.py"
                "${results_folder}/${i}/Progress/f7.txt"
        COMMENT "Analyzing spectra for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_CUSTOM_HYBRID})."
        )
    add_custom_target("custom_${i}_f8" ALL DEPENDS "${results_folder}/${i}/Progress/f8.txt")
    add_dependencies("custom_${i}_f8" "custom_${i}_f7")
        
    #----------------------------------------------------------------------------#
    # Plot spectra
    #----------------------------------------------------------------------------#
    # Define plot names
    set(plot_names "${results_folder}/${i}/Spectra/yspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/mtspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/ptspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/v2spectra.pdf" ;
                   "${results_folder}/${i}/Spectra/meanmt0_midrapidity.pdf" ;
                   "${results_folder}/${i}/Spectra/meanpt_midrapidity.pdf" ;
                   "${results_folder}/${i}/Spectra/midrapidity_yield.pdf" ;
                   "${results_folder}/${i}/Spectra/total_multiplicity.pdf")

    set(plots "")
    foreach(j ${plot_names})
       list(APPEND plots "${j}")
    endforeach(j)

    list(APPEND all_plots ${plots})

    # Perform plotting
    add_custom_command(
         OUTPUT ${plots}
                "${results_folder}/${i}/Progress/f9.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_spectra.py"
                "--input_files" "${results_folder}/${i}/Spectra/*.txt"
                "--Nevents" "${NUM_EVENTS_AFTERBURNER_CUSTOM}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f9.txt"
        DEPENDS "${results_folder}/${i}/Progress/f8.txt"
        COMMENT "Plotting spectra for custom hybrid run."
        )
    add_custom_target("custom_${i}_f9" ALL DEPENDS "${results_folder}/${i}/Progress/f9.txt")
    add_dependencies("custom_${i}_f9" "custom_${i}_f8")
  endforeach(i)

  #----------------------------------------------------------------------------#
  # Average previously obtained spectra
  #----------------------------------------------------------------------------#
  # subdir for tracking simulation progress and description of it
  file(MAKE_DIRECTORY "${results_folder}/Progress")
  FILE(WRITE ${results_folder}/Progress/readme.txt
  "This folder contains empty files created after finishing each step of simulation toolchain.\n"
  "Removing them will force to rebuild a given part of toolchain. No results of simulation are\n"
  "stored here.")
  file(MAKE_DIRECTORY "${results_folder}/Averaged_Spectra")
  set(averaged_spectra "${results_folder}/pT.txt" ;
                       "${results_folder}/mT.txt" ;
                       "${results_folder}/dNdy.txt"
                     )
  add_custom_command(
       OUTPUT ${averaged_spectra}
              "${results_folder}/Progress/f10.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/average_spectra.py"
              "--smash_ana_dir" "${SMASH_PACK_ROOT}/share/smashAnalysis"
              "--pT_files" "${results_folder}/*/Spectra/ptspectra.txt"
              "--mT_files" "${results_folder}/*/Spectra/mtspectra.txt"
              "--y_files" "${results_folder}/*/Spectra/yspectra.txt"
              "--midyY_files" "${results_folder}/*/Spectra/midrapidity_yield.txt"
              "--meanpT_files" "${results_folder}/*/Spectra/meanpt_midrapidity.txt"
              "--v2_files" "${results_folder}/*/Spectra/v2spectra.txt"
              "--Nevents" "${NUM_EVENTS_AFTERBURNER_CUSTOM}"
              "--output_dir" "${results_folder}/Averaged_Spectra/"
      COMMAND "touch" "${results_folder}/Progress/f10.txt"
      COMMENT "Averaging spectra for custom hybrid run."
      )
  add_custom_target("custom_f10" ALL DEPENDS "${results_folder}/Progress/f10.txt")

  #----------------------------------------------------------------------------#
  # Plot averaged quantities
  #----------------------------------------------------------------------------#
  set(averaged_plots "${results_folder}/Averaged_Spectra/yspectra.pdf" ;
                     "${results_folder}/Averaged_Spectra/mtspectra.pdf" ;
                     "${results_folder}/Averaged_Spectra/ptspectra.pdf")

  # Perform plotting of averaged quantities
  add_custom_command(
       OUTPUT ${averaged_plots}
              "${results_folder}/Progress/f11.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_spectra.py"
              "--input_files" "${results_folder}/Averaged_Spectra/*.txt"
              "--Nevents" "${NUM_EVENTS_AFTERBURNER_CUSTOM}"
              "--average" "True"
      COMMAND "touch" "${results_folder}/Progress/f11.txt"
      DEPENDS "${results_folder}/Progress/f10.txt"
      COMMENT "Plotting averaged spectra for custom hybrid run."
      )
  add_custom_target("custom_f11" ALL DEPENDS "${results_folder}/Progress/f11.txt")
  add_dependencies("custom_f11" "custom_f10")
    
  # Define subtargets to enable separated running of the hybrid submodules
  set(target custom_IC)
  add_custom_target(${target} ALL DEPENDS ${smash_IC_files})

  set(target custom_hydro)
  add_custom_target(${target} ALL DEPENDS ${vhlle_freezeout_hypersurfaces})

  set(target custom_sampler)
  add_custom_target(${target} ALL DEPENDS ${sampled_particle_lists})

  set(target custom_afterburner)
  add_custom_target(${target} ALL DEPENDS ${final_particle_lists})

  set(target custom_analysis)
  add_custom_target(${target} ALL DEPENDS ${all_analysis_outputs})

  set(target custom_plots)
  add_custom_target(${target} ALL DEPENDS ${all_plots})

  # Averaging
  set(target custom_average_spectra)
  add_custom_target(${target} ALL DEPENDS ${averaged_spectra})

  set(target custom_average_plots)
  add_custom_target(${target} ALL DEPENDS ${averaged_plots})
endfunction()

################################################################################
#
# Add functionality for a low statistics test case
# This test case is not suitable for production runs, as quanutum number
# conservation cannot be guaranteed owing to the lack of precision.
#
################################################################################
function(test_hybrid
        energy_and_system
        )
  list(GET energy_and_system 0 energy)
  list(GET energy_and_system 1 system)
  message(STATUS "Configuring hybrid test run, not intended for production runs.")
  set(results_folder "${CMAKE_CURRENT_BINARY_DIR}/Hybrid_Results/test")
  file(MAKE_DIRECTORY ${results_folder})

  set(smash_IC_files "")
  set(vhlle_freezeout_hypersurfaces "")
  set(sampled_particle_lists "")
  set(final_particle_lists "")
  set(all_analysis_outputs "")
  set(all_plots "")

  foreach(i RANGE 1 ${NUM_RUNS_TEST_HYBRID})
    file(MAKE_DIRECTORY "${results_folder}/${i}/IC")       # subdir for initial conditions
    file(MAKE_DIRECTORY "${results_folder}/${i}/Hydro")    # subdir for hydro run
    file(MAKE_DIRECTORY "${results_folder}/${i}/Sampler")  # subdir for particle sampling
    file(MAKE_DIRECTORY "${results_folder}/${i}/Afterburner")  # subdir for afterburner
    file(MAKE_DIRECTORY "${results_folder}/${i}/Spectra")  # subdir for analyzed spectra and plots
    # subdir for tracking simulation progress and description of it
    file(MAKE_DIRECTORY "${results_folder}/${i}/Progress")
    FILE(WRITE ${results_folder}/${i}/Progress/readme.txt
    "This folder contains empty files created after finishing each step of simulation toolchain.\n"
    "Removing them will force to rebuild a given part of toolchain. No results of simulation are\n"
    "stored here.")

    set(smash_IC_file "${results_folder}/${i}/IC/SMASH_IC.dat")
    set(smash_IC_oscar "${results_folder}/${i}/IC/SMASH_IC.oscar")
    set(smash_IC_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/smash_initial_conditions_AuAu.yaml")
    set(vhlle_default_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/vhlle_hydro")
    set(vhlle_output_dir "${results_folder}/${i}/Hydro/")
    set(vhlle_updated_config "${results_folder}/${i}/Hydro/vhlle_config")
    set(vhlle_freezeout_hypersurface "${results_folder}/${i}/Hydro/freezeout.dat")
    set(sampler_input_hypersurface "${results_folder}/${i}/Sampler/freezeout.dat")
    set(sampler_default_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/hadron_sampler")
    set(sampler_updated_config "${results_folder}/${i}/Sampler/sampler_config")
    set(sampler_dir "${results_folder}/${i}/Sampler")
    set(sampled_particle_list "${results_folder}/${i}/Sampler/particle_lists.oscar")
    set(full_particle_list "${results_folder}/${i}/Sampler/sampling0")
    set(smash_afterburner_config "${SMASH_PACK_ROOT}/share/smashVHLLEhybrid/configs/smash_afterburner.yaml")
    set(final_particle_list "${results_folder}/${i}/Afterburner/particles_binary.bin")

    list(APPEND smash_IC_files ${smash_IC_file})
    list(APPEND vhlle_freezeout_hypersurfaces ${vhlle_freezeout_hypersurface})
    list(APPEND sampled_particle_lists ${sampled_particle_list})
    list(APPEND final_particle_lists ${final_particle_list})

    #----------------------------------------------------------------------------#
    # Run SMASH and generate initial conditions output
    #----------------------------------------------------------------------------#
    add_custom_command(
         OUTPUT "${smash_IC_file}"
                "${results_folder}/${i}/IC/config.yaml"
                "${results_folder}/${i}/Progress/f1.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/smash"
                "-o" "${results_folder}/${i}/IC"
                "-i" "${smash_IC_config}"
                "-f"
                "-n"
                ">" "${results_folder}/${i}/IC/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f1.txt"
        DEPENDS "${smash_IC_config}"
                "${SMASH_PACK_ROOT}/bin/smash"
        COMMENT "Running SMASH for initial conditions of hybrid test run for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f1" ALL DEPENDS "${results_folder}/${i}/Progress/f1.txt")

    #----------------------------------------------------------------------------#
    # Feed SMASH initial conditions into vHLLE and run hydro evolution
    #----------------------------------------------------------------------------#
    # create input file with correct paths for vHLLE
    add_custom_command(
         OUTPUT "${vhlle_updated_config}"
                "${results_folder}/${i}/Progress/f2.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_vhlle_config.py"
                "--vhlle_config" "${vhlle_default_config}"
                "--smash_ic" "${smash_IC_file}"
                "--output_file" "${vhlle_updated_config}"
                "--test"
        COMMAND "touch" "${results_folder}/${i}/Progress/f1.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_vhlle_config.py"
                "${vhlle_default_config}"
                "${results_folder}/${i}/Progress/f1.txt"
        COMMENT "Creating input file for vHLLE for hybrid test run for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f2" ALL DEPENDS "${results_folder}/${i}/Progress/f2.txt")
    add_dependencies("test_${system}_${energy}_${i}_f2" "test_${system}_${energy}_${i}_f1")

    # Run hydro
    add_custom_command(
         OUTPUT "${vhlle_freezeout_hypersurface}"
                "${results_folder}/${i}/Progress/f3.txt"
        COMMAND "${VHLLE_ROOT}/bin/hlle_visc"
                "-params" "${vhlle_updated_config}"
                "-ISinput" "${smash_IC_file}"
                "-outputDir" "${vhlle_output_dir}"
                "-configDir" "${VHLLE_PARAMS}"
                ">" "${vhlle_output_dir}/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f3.txt"
        DEPENDS "${vhlle_updated_config}"
                "${VHLLE_ROOT}/bin/hlle_visc"
                "${results_folder}/${i}/Progress/f2.txt"
        COMMENT "Running vHLLE for hybrid test run for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f3" ALL DEPENDS "${results_folder}/${i}/Progress/f3.txt")
    add_dependencies("test_${system}_${energy}_${i}_f3" "test_${system}_${energy}_${i}_f2")

    #----------------------------------------------------------------------------#
    # Run Cooper-Frye sampler for particlization of freezeout surface
    #----------------------------------------------------------------------------#
    # create input file with correct paths for sampler
    add_custom_command(
         OUTPUT "${sampler_updated_config}"
                "${results_folder}/${i}/Progress/f4.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_sampler_config.py"
                "--sampler_config" "${sampler_default_config}"
                "--vhlle_config" "${vhlle_updated_config}"
                "--output_file" "${sampler_updated_config}"
                "--Nevents" "${NUM_EVENTS_AFTERBURNER_TEST}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f4.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/create_sampler_config.py"
                "${sampler_default_config}"
                "${results_folder}/${i}/Progress/f3.txt"
        COMMENT "Creating input file for hybrid test run for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f4" ALL DEPENDS "${results_folder}/${i}/Progress/f4.txt")
    add_dependencies("test_${system}_${energy}_${i}_f4" "test_${system}_${energy}_${i}_f3")

    # Run sampler
    add_custom_command(
         OUTPUT ${sampled_particle_list}
                "${results_folder}/${i}/Progress/f5.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/hadron_sampler" "events" "1" "${sampler_updated_config}"
                ">" "${results_folder}/${i}/Sampler/Terminal_Output.txt"
        COMMAND "touch" "${results_folder}/${i}/Progress/f5.txt"
        DEPENDS "${SMASH_PACK_ROOT}/bin/hadron_sampler"
                "${sampler_updated_config}"
                "${results_folder}/${i}/Progress/f4.txt"
        COMMENT "Sampling particles from freezeout hypersurface (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f5" ALL DEPENDS "${results_folder}/${i}/Progress/f5.txt")
    add_dependencies("test_${system}_${energy}_${i}_f5" "test_${system}_${energy}_${i}_f4")

    #----------------------------------------------------------------------------#
    # Run SMASH as afterburner
    #----------------------------------------------------------------------------#
    # Add spectators to particle list and rename it to be in accordance
    # with SMASH list modus input format
    add_custom_command(
         OUTPUT ${full_particle_list}
                "${results_folder}/${i}/Progress/f6.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/add_spectators.py"
                "--sampled_particle_list" "${sampler_dir}/particle_lists.oscar"
                "--initial_particle_list" "${smash_IC_oscar}"
                "--output_file" "${full_particle_list}"
                "--smash_config" "${results_folder}/${i}/IC/config.yaml"
        COMMAND "touch" "${results_folder}/${i}/Progress/f6.txt"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/add_spectators.py"
                "${results_folder}/${i}/Progress/f5.txt"
        COMMENT "Adding spectators to sampled particle list for afterburner (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f6" ALL DEPENDS "${results_folder}/${i}/Progress/f6.txt")
    add_dependencies("test_${system}_${energy}_${i}_f6" "test_${system}_${energy}_${i}_f5")

    # Run afterburner evolution
    add_custom_command(
         OUTPUT "${final_particle_list}"
                "${results_folder}/${i}/Afterburner/config.yaml"
                "${results_folder}/${i}/Progress/f7.txt"
        COMMAND "${SMASH_PACK_ROOT}/bin/smash"
                "-i" "${smash_afterburner_config}"
                "-c" "Modi: { List: { File_Directory: ${sampler_dir}} }"
                "-c" "General: { Nevents: ${NUM_EVENTS_AFTERBURNER_TEST} }"
                "-o" "${results_folder}/${i}/Afterburner"
                "-f"
                "-n"
                ">" "/dev/null"
        COMMAND "touch" "${results_folder}/${i}/Progress/f7.txt"
        DEPENDS "${smash_afterburner_config}"
                "${SMASH_PACK_ROOT}/bin/smash"
                "${full_particle_list}"
                "${results_folder}/${i}/Progress/f6.txt"
        COMMENT "Running SMASH afterburner for hybrid test run for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f7" ALL DEPENDS "${results_folder}/${i}/Progress/f7.txt")
    add_dependencies("test_${system}_${energy}_${i}_f7" "test_${system}_${energy}_${i}_f6")

    #----------------------------------------------------------------------------#
    # Analyse data
    #----------------------------------------------------------------------------#
    # Define filenames for particle spectra and create list of those
    set(spectra_fnames "${results_folder}/${i}/Spectra/yspectra.txt" ;
                       "${results_folder}/${i}/Spectra/mtspectra.txt" ;
                       "${results_folder}/${i}/Spectra/ptspectra.txt" ;
                       "${results_folder}/${i}/Spectra/v2spectra.txt" ;
                       "${results_folder}/${i}/Spectra/meanmt0_midrapidity.txt" ;
                       "${results_folder}/${i}/Spectra/meanpt_midrapidity.txt" ;
                       "${results_folder}/${i}/Spectra/midrapidity_yield.txt" ;
                       "${results_folder}/${i}/Spectra/total_multiplicity.txt")
    set(analysis_outputs "")
    foreach(j ${spectra_fnames})
        list(APPEND analysis_outputs "${j}")
    endforeach(j)

    list(APPEND all_analysis_outputs ${analysis_outputs})

    # Perform analysis
    add_custom_command(
         OUTPUT ${analysis_outputs}
                "${results_folder}/${i}/Progress/f8.txt"
        COMMAND "python3" "${SMASH_PACK_ROOT}/share/smashAnalysis/test/energy_scan/mult_and_spectra.py"
                "--output_files" ${analysis_outputs}
                "--input_files" "${final_particle_list}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f8.txt"
        DEPENDS "${SMASH_PACK_ROOT}/share/smashAnalysis/test/energy_scan/mult_and_spectra.py"
                "${results_folder}/${i}/Progress/f7.txt"
        COMMENT "Analyzing spectra for ${system} @ ${energy} GeV (${i}/${NUM_RUNS_TEST_HYBRID})."
        )
    add_custom_target("test_${system}_${energy}_${i}_f8" ALL DEPENDS "${results_folder}/${i}/Progress/f8.txt")
    add_dependencies("test_${system}_${energy}_${i}_f8" "test_${system}_${energy}_${i}_f7")
        
    #----------------------------------------------------------------------------#
    # Plot spectra
    #----------------------------------------------------------------------------#
    # Define plot names
    set(plot_names "${results_folder}/${i}/Spectra/yspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/mtspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/ptspectra.pdf" ;
                   "${results_folder}/${i}/Spectra/v2spectra.pdf" ;
                   "${results_folder}/${i}/Spectra/meanmt0_midrapidity.pdf" ;
                   "${results_folder}/${i}/Spectra/meanpt_midrapidity.pdf" ;
                   "${results_folder}/${i}/Spectra/midrapidity_yield.pdf" ;
                   "${results_folder}/${i}/Spectra/total_multiplicity.pdf")

    set(plots "")
    foreach(j ${plot_names})
       list(APPEND plots "${j}")
    endforeach(j)

    list(APPEND all_plots ${plots})

    # Perform plotting
    add_custom_command(
         OUTPUT ${plots}
                "${results_folder}/${i}/Progress/f9.txt"
        COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_spectra.py"
                "--input_files" "${results_folder}/${i}/Spectra/*.txt"
                "--Nevents" "${NUM_EVENTS_AFTERBURNER_TEST}"
        COMMAND "touch" "${results_folder}/${i}/Progress/f9.txt"
        DEPENDS "${results_folder}/${i}/Progress/f8.txt"
        COMMENT "Plotting spectra for custom hybrid run."
        )
    add_custom_target("test_${system}_${energy}_${i}_f9" ALL DEPENDS "${results_folder}/${i}/Progress/f9.txt")
    add_dependencies("test_${system}_${energy}_${i}_f9" "test_${system}_${energy}_${i}_f8")
  endforeach(i)

  #----------------------------------------------------------------------------#
  # Average previously obtained spectra
  #----------------------------------------------------------------------------#
  # subdir for tracking simulation progress and description of it
  file(MAKE_DIRECTORY "${results_folder}/Progress")
  FILE(WRITE ${results_folder}/Progress/readme.txt
  "This folder contains empty files created after finishing each step of simulation toolchain.\n"
  "Removing them will force to rebuild a given part of toolchain. No results of simulation are\n"
  "stored here.")
  file(MAKE_DIRECTORY "${results_folder}/Averaged_Spectra")
  set(averaged_spectra "${results_folder}/pT.txt" ;
                       "${results_folder}/mT.txt" ;
                       "${results_folder}/dNdy.txt"
                     )
  add_custom_command(
       OUTPUT ${averaged_spectra}
              "${results_folder}/Progress/f10.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/average_spectra.py"
              "--smash_ana_dir" "${SMASH_PACK_ROOT}/share/smashAnalysis"
              "--pT_files" "${results_folder}/*/Spectra/ptspectra.txt"
              "--mT_files" "${results_folder}/*/Spectra/mtspectra.txt"
              "--y_files" "${results_folder}/*/Spectra/yspectra.txt"
              "--midyY_files" "${results_folder}/*/Spectra/midrapidity_yield.txt"
              "--meanpT_files" "${results_folder}/*/Spectra/meanpt_midrapidity.txt"
              "--v2_files" "${results_folder}/*/Spectra/v2spectra.txt"
              "--Nevents" "${NUM_EVENTS_AFTERBURNER_TEST}"
              "--output_dir" "${results_folder}/Averaged_Spectra/"
      COMMAND "touch" "${results_folder}/Progress/f10.txt"
      COMMENT "Averaging spectra for hybrid test run for ${system} @ ${energy} GeV."
      )
  add_custom_target("test_f10" ALL DEPENDS "${results_folder}/Progress/f10.txt")

  #----------------------------------------------------------------------------#
  # Plot averaged quantities
  #----------------------------------------------------------------------------#
  set(averaged_plots "${results_folder}/Averaged_Spectra/yspectra.pdf" ;
                     "${results_folder}/Averaged_Spectra/mtspectra.pdf" ;
                     "${results_folder}/Averaged_Spectra/ptspectra.pdf")

  # Perform plotting of averaged quantities
  add_custom_command(
       OUTPUT ${averaged_plots}
              "${results_folder}/Progress/f11.txt"
      COMMAND "python3" "${CMAKE_CURRENT_SOURCE_DIR}/python_scripts/plot_spectra.py"
              "--input_files" "${results_folder}/Averaged_Spectra/*.txt"
              "--Nevents" "${NUM_EVENTS_AFTERBURNER_TEST}"
              "--average" "True"
      COMMAND "touch" "${results_folder}/Progress/f11.txt"
      DEPENDS "${results_folder}/Progress/f10.txt"
      COMMENT "Plotting averaged spectra for hybrid test run for ${system} @ ${energy} GeV."
      )
  add_custom_target("test_f11" ALL DEPENDS "${results_folder}/Progress/f11.txt")
  add_dependencies("test_f11" "test_f10")
    
  # Define subtargets to enable separated running of the hybrid submodules
  set(target test_IC)
  add_custom_target(${target} ALL DEPENDS ${smash_IC_files})

  set(target test_hydro)
  add_custom_target(${target} ALL DEPENDS ${vhlle_freezeout_hypersurfaces})

  set(target test_sampler)
  add_custom_target(${target} ALL DEPENDS ${sampled_particle_lists})

  set(target test_afterburner)
  add_custom_target(${target} ALL DEPENDS ${final_particle_lists})

  set(target test_analysis)
  add_custom_target(${target} ALL DEPENDS ${all_analysis_outputs})

  set(target test_plots)
  add_custom_target(${target} ALL DEPENDS ${all_plots})

  # Averaging
  set(target test_average_spectra)
  add_custom_target(${target} ALL DEPENDS ${averaged_spectra})

  set(target test_average_plots)
  add_custom_target(${target} ALL DEPENDS ${averaged_plots})
endfunction()

if (SMASH_ANALYSES)
  get_filename_component(smashAnalysesFile "${SMASH_ANALYSES}" REALPATH BASE_DIR "${CMAKE_BINARY_DIR}") # get absolute path
  include(${smashAnalysesFile})
else()
  MESSAGE(FATAL_ERROR "You have to set file with analysis to run. Use -DSMASH_ANALYSES=/some/file/name")
endif()
