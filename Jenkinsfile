@Library('xmos_jenkins_shared_library@v0.14.2') _

getApproval()

pipeline {
  agent none
  parameters {
    string(
      name: 'TOOLS_VERSION',
      defaultValue: '15.0.2',
      description: 'The tools version to build with (check /projects/tools/ReleasesTools/)'
    )
  }
  stages {
    stage('Standard build and XS2 tests') {
      agent {
        label 'x86_64&&brew'
      }
      environment {
        REPO = 'lib_dsp'
        VIEW = getViewName(REPO)
      }
      options {
        skipDefaultCheckout()
      }
      stages {
        stage('Get view') {
          steps {
            xcorePrepareSandbox("${VIEW}", "${REPO}")
          }
        }
        stage('Library checks') {
          steps {
            xcoreLibraryChecks("${REPO}")
          }
        }
        stage('Tests') {
          stages {
            stage('Test Biquad') {
              steps {
                dir("${REPO}/tests/test_biquad") {
                  runWaf('.')
                  viewEnv() {
                    runPytest()
                  }
                }
              }
            }
            stage("Unit tests") {
              steps {
                dir("${REPO}/tests/dsp_unit_tests") {
                  runWaf('.')
                  viewEnv() {
                    runPytest()
                  }
                }
              }
            }
            stage("Legacy Tests") {
              steps {
                runXmostest("${REPO}", 'tests')
              }
            }
          }
        }
        stage('Build') {
          steps {
            dir("${REPO}") {
              /* Cannot call xcoreAppNoteBuild('AN00209_xCORE-200_DSP_Library')
               * due to the use of multiple applications within this app note.
               */
              xcoreAllAppsBuild('AN00209_xCORE-200_DSP_Library')
              dir('AN00209_xCORE-200_DSP_Library') {
                runXdoc('doc')
              }
              dir("${REPO}") {
                runXdoc('doc')
              }
            }
          }
        }
        stage('Build XCOREAI') {
          steps {
            dir("${REPO}") {
              // Build these individually (or we can extend xcoreAllAppsBuild to support an argument
              dir('AN00209_xCORE-200_DSP_Library/') {
                script {
                  apps = sh(script: 'find . -maxdepth 1 -name app* | cut -c 3-', returnStdout: true).trim().split("\\r?\\n")
                  apps.each() {
                    dir(it) {
                      runXmake(".", "", "XCOREAI=1")
                      stash name: it, includes: 'bin/*xcoreai/*.xe, '
                    }
                  }
                }
              }

              // Build Tests
              dir('tests/') {
                script {
                  tests = [
                    "test_fft_forward_real",
                    "test_fft_inverse_blank_forward"
                  ]
                  tests.each() {
                    dir(it) {
                      runXmake(".", "", "XCOREAI=1")
                      stash name: it, includes: 'bin/*xcoreai/*.xe, '
                    }
                  }
                }
              }
            }
          }
        }
      } // stages
      post {
        cleanup {
          xcoreCleanSandbox()
        }
      }
    } // Stage Standard Build

    stage('xcore.ai Verification'){
      agent {
        label 'xcore.ai-explorer'
      }
      environment {
        // '/XMOS/tools' from get_tools.py and rest from tools installers
        TOOLS_PATH = "/XMOS/tools/${params.TOOLS_VERSION}/XMOS/xTIMEcomposer/${params.TOOLS_VERSION}"
      }
      stages{
        stage('Install Dependencies') {
          steps {
            sh '/XMOS/get_tools.py ' + params.TOOLS_VERSION
            installDependencies()
          }
        }
        stage('xrun'){
          steps{
            dir("${REPO}") {
              toolsEnv(TOOLS_PATH) {  // load xmos tools
                dir('AN00209_xCORE-200_DSP_Library/') {
                  // Unstash all XCOREAI App notes
                  script {
                    apps = sh(script: 'find . -maxdepth 1 -name app* | cut -c 3-', returnStdout: true).trim().split("\\r?\\n")
                    apps.each() {
                      dir(it) {
                        unstash it
                      }
                    }
                  }
                  // Run all the tests
                  // app_adaptive - expect
                  sh 'xrun --io --id 0 app_adaptive/bin/xcoreai/app_adaptive.xe &> app_adaptive_test.txt'
                  sh 'cat app_adaptive_test.txt && diff app_adaptive_test.txt ../tests/adaptive_test.expect'

                  // app_atan2_hypot - no test
                  sh 'xrun --io --id 0 app_atan2_hypot/bin/xcoreai/app_atan2_hypot.xe'

                  // app_bfp - expect
                  sh 'xrun --io --id 0 app_bfp/bin/xcoreai/app_bfp.xe &> app_bfp_test.txt'
                  sh 'cat app_bfp_test.txt && diff app_bfp_test.txt ../tests/bfp_test.expect'

                  // app_complex - expect
                  sh 'xrun --io --id 0 app_complex/bin/xcoreai/app_complex.xe &> app_complex_test.txt'
                  sh 'cat app_complex_test.txt && diff app_complex_test.txt ../tests/complex_test.expect'

                  // app_complex_fir - expect
                  sh 'xrun --io --id 0 app_complex_fir/bin/xcoreai/app_complex_fir.xe &> app_complex_fir_test.txt'
                  sh 'cat app_complex_fir_test.txt && diff app_complex_fir_test.txt ../tests/complex_fir_test.expect'
                  // app_dct - no test
                  sh 'xrun --io --id 0 app_dct/bin/xcoreai/app_dct.xe'

                  // app_design - expect
                  sh 'xrun --io --id 0 app_design/bin/xcoreai/app_design.xe &> app_design_test.txt'
                  sh 'cat app_design_test.txt && diff app_design_test.txt ../tests/design_test.expect'

                  // app_fft - no test
                  sh 'xrun --io --id 0 app_fft/bin/xcoreai/app_fft.xe'

                  // app_fft_dif - no test
                  sh 'xrun --io --id 0 app_fft_dif/bin/xcoreai/app_fft_dif.xe'

                  // app_fft_double_buf - no test
                  sh 'xrun --io --id 0 app_fft_double_buf/bin/xcoreai/app_fft_double_buf.xe'

                  // app_fft_real_single - expect
                  sh 'xrun --io --id 0 app_fft_real_single/bin/xcoreai/app_fft_real_single.xe &> app_fft_real_single_test.txt'
                  sh 'cat app_fft_real_single_test.txt && diff app_fft_real_single_test.txt ../tests/fft_real_single_test.expect'

                  // app_fft_timing - no test
                  sh 'xrun --io --id 0 app_fft_timing/bin/xcoreai/app_fft_timing.xe'

                  // app_filters - expect
                  sh 'xrun --io --id 0 app_filters/bin/xcoreai/app_filters.xe &> app_filters_test.txt'
                  sh 'cat app_filters_test.txt && diff app_filters_test.txt ../tests/filters_test.expect'

                  // app_math - expect
                  sh 'xrun --io --id 0 app_math/bin/xcoreai/app_math.xe &> app_math_test.txt'
                  sh 'cat app_math_test.txt && diff app_math_test.txt ../tests/math_test.expect'

                  // app_matrix - expect
                  sh 'xrun --io --id 0 app_matrix/bin/xcoreai/app_matrix.xe &> app_matrix_test.txt'
                  sh 'cat app_matrix_test.txt && diff app_matrix_test.txt ../tests/matrix_test.expect'

                  // app_statistics - expect
                  sh 'xrun --io --id 0 app_statistics/bin/xcoreai/app_statistics.xe &> app_statistics_test.txt'
                  sh 'cat app_statistics_test.txt && diff app_statistics_test.txt ../tests/statistics_test.expect'

                  // app_vector - expect
                  sh 'xrun --io --id 0 app_vector/bin/xcoreai/app_vector.xe &> app_vector_test.txt'
                  sh 'cat app_vector_test.txt && diff app_vector_test.txt ../tests/vector_test.expect'

                  // app_window_post_fft - expect (test_hann)
                  sh 'xrun --io --id 0 app_window_post_fft/bin/xcoreai/app_window_post_fft.xe &> app_window_post_fft_test.txt'
                  sh 'cat app_window_post_fft_test.txt && diff app_window_post_fft_test.txt ../tests/test_hann.expect'
                }

                //Run this and diff against expected output. Note we have the lib files here available


                ////Run this and diff against expected output. Note we have the lib files here available
                //unstash 'debug_printf_test'
                //sh 'xrun --io --id 0 bin/xcoreai/debug_printf_test.xe &> debug_printf_test.txt'
                //sh 'cat debug_printf_test.txt && diff debug_printf_test.txt tests/test.expect'

                ////Just run these and error on exception
                //unstash 'AN00239'
                //sh 'xrun --io --id 0 bin/xcoreai/AN00239.xe'
                //unstash 'app_debug_printf'
                //sh 'xrun --io --id 0 bin/xcoreai/app_debug_printf.xe'
              }
            }
          }
        }
      }//stages
      post {
        cleanup {
          cleanWs()
        }
      }
    }// xcore.ai


  }
  post {
    success {
      updateViewfiles()
    }
    cleanup {
      xcoreCleanSandbox()
    }
  }
}
