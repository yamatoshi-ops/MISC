c/
  main.c
  controller_phase7.h
  controller_phase7_optional_debug.h
  phase7_types.h
  rte_phase7.{c,h}
  controller_app_phase7.{c,h}
  periodic_task_phase7.{c,h}

  common/
    torque_cmd_lpf.*
    torque_reference.*
    feedback_transform.*
    torque_estimation.*
    current_control.*
    vdq_feed_forward.*
    voltage_limit.*
    pwm_duty.*
    carrier_command.*
    flux_map.*

  ipm/
    no8_ipm_peak_bottom_phase7.*
    no8_ipm_periodic_phase7.*

  ipm-ekf/
    no8_ipm_ekf_peak_bottom_phase7.*
    no8_ipm_ekf_periodic_phase7.*
    ekf_delta_phi_monitor.*

  ipm-kf/
    no8_ipm_kf_peak_bottom_phase7.*
    no8_ipm_kf_periodic_phase7.*
    qs_kf_monitor.*

  im-cvc/
    no8_im_peak_bottom_phase7.*
    no8_im_periodic_phase7.*
    no8_im_cvc_constants.*
    im_cvc_reference.*
    im_cvc_angle.*
    im_flux_observer.*
    im_rr_adaptation.*

  custom/
    rte_phase7_custom.*
    no8_ipm_custom_peak_bottom_phase7.*
    no8_ipm_custom_periodic_phase7.*
    no8_ipm_custom_user.*
