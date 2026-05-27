<?php
/**
 * Plugin Name: HIGHEST DATA PLUG
 * Description: v5.6.0 – Moolre SMS provider + Moolre payment gateway (toggleable, redirects walk-in & agent flows; USSD stays on Paystack MoMo).
 * Version: 5.6.0
 * Author: Built for Poku
 */
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

class HighestDataPlug_Plugin {

    public $emergency_sms_enabled;
    public $emergency_sms_phone;
    public $emergency_prompt_enabled;   // new: prompt agents on order failure

    public function __construct() {
        $this->emergency_sms_enabled = get_option( 'highest_emergency_sms_enabled', 'yes' ) === 'yes';
        $this->emergency_sms_phone   = get_option( 'highest_emergency_sms_phone', '' );
        $this->emergency_prompt_enabled = get_option( 'highest_emergency_prompt_enabled', 'yes' ) === 'yes'; // new toggle
        $this->maybe_add_balance_after_column();

        add_action( 'after_setup_theme', function () {
            if ( is_user_logged_in() && current_user_can( 'agent' ) ) {
                show_admin_bar( false );
            }
        } );
        add_action( 'init', function () {
            if (
                ! is_user_logged_in() &&
                isset( $_SERVER['REQUEST_METHOD'] ) && $_SERVER['REQUEST_METHOD'] === 'GET' &&
                isset( $_SERVER['REQUEST_URI'] ) &&
                strpos( $_SERVER['REQUEST_URI'], 'wp-login.php' ) !== false
            ) {
                wp_redirect( site_url( '/agent-login' ) );
                exit;
            }
        } );

        add_shortcode( 'waec_buy', array( $this, 'render_waec_buy' ) );
        add_shortcode( 'waec_tracker', array( $this, 'render_waec_tracker' ) );
        add_action( 'wp_ajax_highest_waec_purchase', array( $this, 'ajax_waec_purchase' ) );
        add_action( 'wp_ajax_nopriv_highest_waec_purchase', array( $this, 'ajax_waec_purchase' ) );
        add_action( 'wp_ajax_highest_waec_retrieve', array( $this, 'ajax_waec_retrieve' ) );
        add_action( 'wp_ajax_nopriv_highest_waec_retrieve', array( $this, 'ajax_waec_retrieve' ) );
        add_shortcode( 'highestdataplug_fullsite', array( $this, 'render_fullsite' ) );
        add_shortcode( 'highestdataplug_agent_portal', array( $this, 'render_agent_portal' ) );
        add_shortcode( 'agent_login_form', array( $this, 'render_agent_login_form' ) );
        add_action( 'wp_enqueue_scripts', array( $this, 'enqueue_assets' ) );
        add_action( 'admin_menu', array( $this, 'add_admin_menu' ) );

        // AJAX handlers
        add_action( 'wp_ajax_highest_register_agent', array( $this, 'ajax_register_agent' ) );
        add_action( 'wp_ajax_nopriv_highest_register_agent', array( $this, 'ajax_register_agent' ) );
        add_action( 'wp_ajax_highest_buy_data', array( $this, 'ajax_buy_data' ) );
        add_action( 'wp_ajax_nopriv_highest_buy_data', array( $this, 'ajax_buy_data' ) );
        // Nonce-free, Paystack-verified order creation (replaces nonce-based handlers)
        add_action( 'wp_ajax_highest_buy_data_verified',        array( $this, 'ajax_buy_data_verified' ) );
        add_action( 'wp_ajax_nopriv_highest_buy_data_verified', array( $this, 'ajax_buy_data_verified' ) );
        add_action( 'wp_ajax_highest_save_agent_prices', array( $this, 'ajax_save_agent_prices' ) );
        add_action( 'wp_ajax_highest_save_storefront', array( $this, 'ajax_save_storefront' ) );
        add_action( 'wp_ajax_highest_agent_buy_data', array( $this, 'ajax_agent_buy_data' ) );
        add_action( 'wp_ajax_highest_get_orders', array( $this, 'ajax_get_orders' ) );
        add_action( 'wp_ajax_highest_track_order', array( $this, 'ajax_track_order' ) );
        add_action( 'wp_ajax_highest_verify_topup', array( $this, 'ajax_verify_topup' ) );
        add_action( 'wp_ajax_highest_public_track_order', array( $this, 'ajax_public_track_order' ) );
        add_action( 'wp_ajax_nopriv_highest_public_track_order', array( $this, 'ajax_public_track_order' ) );
        add_action( 'wp_ajax_highest_check_dakazina_status', array( $this, 'ajax_check_dakazina_status' ) );
        add_action( 'wp_ajax_nopriv_highest_check_dakazina_status', array( $this, 'ajax_check_dakazina_status' ) );
        add_action( 'wp_ajax_highest_auto_check_statuses', array( $this, 'ajax_auto_check_statuses' ) );
        add_action( 'wp_ajax_nopriv_highest_auto_check_statuses', array( $this, 'ajax_auto_check_statuses' ) );
        add_action( 'wp_ajax_highest_request_manual_topup', array( $this, 'ajax_request_manual_topup' ) );
        add_action( 'wp_ajax_nopriv_highest_request_manual_topup', array( $this, 'ajax_request_manual_topup' ) );
        add_action( 'wp_ajax_highest_update_profile', array( $this, 'ajax_update_profile' ) );
        add_action( 'wp_ajax_highest_hubtel_get_sender_ids', array( $this, 'ajax_hubtel_get_sender_ids' ) );
        add_action( 'wp_ajax_highest_bulk_sms', array( $this, 'ajax_bulk_sms' ) );
        add_action( 'wp_ajax_highest_bulk_sms_customers', array( $this, 'ajax_bulk_sms_customers' ) );
        add_action( 'wp_ajax_highest_log_payment_initiation', array( $this, 'ajax_log_payment_initiation' ) );
        add_action( 'wp_ajax_nopriv_highest_log_payment_initiation', array( $this, 'ajax_log_payment_initiation' ) );
        add_action( 'wp_ajax_highest_verify_payment_session', array( $this, 'ajax_verify_payment_session' ) );
        add_action( 'wp_ajax_highest_process_payment_session', array( $this, 'ajax_process_payment_session' ) );
        add_action( 'wp_ajax_highest_register_afa', array( $this, 'ajax_register_afa' ) );
        add_action( 'wp_ajax_highest_get_store_attempts', array( $this, 'ajax_get_store_attempts' ) );

        // v4.2.0+ AJAX
        add_action( 'wp_ajax_highest_agent_verify_payment', array( $this, 'ajax_agent_verify_payment' ) );
        add_action( 'wp_ajax_highest_admin_verify_payment', array( $this, 'ajax_admin_verify_payment' ) );
        add_action( 'wp_ajax_highest_get_agent_payment_sessions', array( $this, 'ajax_get_agent_payment_sessions' ) );
        add_action( 'wp_ajax_highest_get_agent_payment_sessions_count', array( $this, 'ajax_get_agent_payment_sessions_count' ) );
        add_action( 'wp_ajax_highest_send_emergency_sms', array( $this, 'ajax_send_emergency_sms' ) );
        add_action( 'wp_ajax_highest_get_dashboard_stats', array( $this, 'ajax_get_dashboard_stats' ) );

        // v5.1: new admin safe verify handlers
        add_action( 'wp_ajax_highest_verify_payment_session_v2', array( $this, 'ajax_verify_payment_session_v2' ) );
        add_action( 'wp_ajax_highest_create_order_for_session', array( $this, 'ajax_create_order_for_session' ) );

        // Referral system AJAX handlers
        add_action( 'wp_ajax_highest_register_subagent', array( $this, 'ajax_register_subagent' ) );
        add_action( 'wp_ajax_highest_get_my_subagents', array( $this, 'ajax_get_my_subagents' ) );
        add_action( 'wp_ajax_highest_save_referral_bonus', array( $this, 'ajax_save_referral_bonus' ) );
        add_action( 'wp_ajax_highest_get_referral_bonus', array( $this, 'ajax_get_referral_bonus' ) );
        add_action( 'wp_ajax_highest_claim_referral', array( $this, 'ajax_claim_referral' ) );
        add_action( 'wp_ajax_highest_get_agent_performance', array( $this, 'ajax_get_agent_performance' ) );
        add_action( 'wp_ajax_highest_send_retry_sms', array( $this, 'ajax_send_retry_sms' ) );
        add_action( 'wp_ajax_highest_notify_agent',   array( $this, 'ajax_notify_agent' ) );
        add_action( 'wp_ajax_highest_get_withdrawal_history', array( $this, 'ajax_get_withdrawal_history' ) );

        // Referral bonus maturation handlers
        add_action( 'wp_ajax_highest_get_locked_bonus_status', array( $this, 'ajax_get_locked_bonus_status' ) );
        add_action( 'wp_ajax_highest_claim_locked_bonuses',    array( $this, 'ajax_claim_locked_bonuses' ) );

        // v5.3: Multi-level sub-agent permissions
        add_action( 'wp_ajax_highest_get_my_subagents_v2',   array( $this, 'ajax_get_my_subagents_v2' ) );
        add_action( 'wp_ajax_highest_update_subagent_perm',  array( $this, 'ajax_update_subagent_perm' ) );

        // Wholesale pricing AJAX handlers
        add_action( 'wp_ajax_highest_get_wholesale_prices',  array( $this, 'ajax_get_wholesale_prices' ) );
        add_action( 'wp_ajax_highest_save_wholesale_prices', array( $this, 'ajax_save_wholesale_prices' ) );

        // Moolre payment checkout URL (used by walk-in + agent JS when gateway = moolre)
        add_action( 'wp_ajax_highest_get_moolre_checkout_url',        array( $this, 'ajax_get_moolre_checkout_url' ) );
        add_action( 'wp_ajax_nopriv_highest_get_moolre_checkout_url', array( $this, 'ajax_get_moolre_checkout_url' ) );

        // Tier upgrade AJAX handlers
        add_action( 'wp_ajax_highest_get_tier_upgrades', array( $this, 'ajax_get_tier_upgrades' ) );
        add_action( 'wp_ajax_highest_upgrade_tier',      array( $this, 'ajax_upgrade_tier' ) );


        add_action( 'init', array( $this, 'handle_dakazina_webhook' ) );
        add_action( 'rest_api_init', array( $this, 'register_paystack_webhook' ) );
        add_action( 'rest_api_init', array( $this, 'register_moolre_webhook' ) );
        add_action( 'rest_api_init', array( $this, 'register_ussd_callback' ) );
        add_action( 'rest_api_init', array( $this, 'register_test_route' ) );
        add_action( 'init', array( $this, 'create_agent_portal_page' ) );
        add_action( 'template_redirect', array( $this, 'redirect_agents_to_portal' ) );
        add_action( 'template_redirect', array( $this, 'handle_agent_storefront' ) );
        add_filter( 'login_redirect', array( $this, 'redirect_agent_login' ), 10, 3 );

        add_action( 'init', array( $this, 'handle_agent_profit_withdrawal' ) );
        add_action( 'admin_init', array( $this, 'handle_withdrawal_approval' ) );
        add_action( 'admin_init', array( $this, 'handle_update_tier' ) );
        add_action( 'admin_init', array( $this, 'handle_force_delivered' ) );
        add_action( 'admin_init', array( $this, 'handle_deduction' ) );
        add_action( 'admin_init', array( $this, 'handle_manual_topup_approval' ) );
        add_action( 'admin_init', array( $this, 'handle_toggle_sms_disable' ) );
        add_action( 'admin_init', array( $this, 'handle_toggle_suspend' ) );
        add_action( 'admin_init', array( $this, 'handle_afa_actions' ) );
        add_action( 'admin_init', array( $this, 'handle_queue_order' ) );
        add_action( 'admin_init', array( $this, 'handle_refund_wallet' ) );

        register_activation_hook( __FILE__, array( $this, 'activate_plugin' ) );

        add_action( 'wp_login', array( $this, 'handle_daily_login_bonus' ), 10, 2 );
        add_action( 'highest_cron_auto_check', array( $this, 'cron_auto_check_orders' ) );
        add_action( 'highest_cron_check_payment_sessions', array( $this, 'cron_check_payment_sessions' ) );
        add_action( 'highest_cron_process_queued', array( $this, 'process_queued_orders' ) );

        if ( ! wp_next_scheduled( 'highest_cron_auto_check' ) ) {
            wp_schedule_event( time(), 'thirty_minutes', 'highest_cron_auto_check' );
        }
        // highest_cron_check_payment_sessions and highest_cron_process_queued
        // have been permanently disabled to prevent duplicate/flood orders.

        // Safe session processor – only handles Paystack-verified sessions (every minute)
        add_action( 'highest_cron_safe_process_sessions', array( $this, 'cron_safe_process_validated_sessions' ) );
        if ( ! wp_next_scheduled( 'highest_cron_safe_process_sessions' ) ) {
            wp_schedule_event( time(), 'every_minute', 'highest_cron_safe_process_sessions' );
        }

        add_action( 'highest_cron_send_scheduled_sms', array( $this, 'cron_send_scheduled_sms' ) );
        if ( ! wp_next_scheduled( 'highest_cron_send_scheduled_sms' ) ) {
            wp_schedule_event( time(), 'every_minute', 'highest_cron_send_scheduled_sms' );
        }
        add_filter( 'cron_schedules', array( $this, 'add_cron_interval' ) );

        // ── Self-healing migration: normalise any NULL/empty status rows ──────
        // Rows inserted before the column had NOT NULL DEFAULT 'active' will
        // break every query that filters on status = 'active'. Run once and
        // record the fact so subsequent requests skip the DB write entirely.
        if ( ! get_option( 'hdp_status_migration_done' ) ) {
            global $wpdb;
            $wpdb->query(
                "UPDATE {$wpdb->prefix}highest_agent_referrals
                 SET status = 'active'
                 WHERE status IS NULL OR status = ''"
            );
            // Also backfill permission columns that were added in v5.3 without a DEFAULT fill.
            $wpdb->query( "UPDATE {$wpdb->prefix}highest_agent_referrals SET allow_upgrade   = 1 WHERE allow_upgrade   IS NULL" );
            $wpdb->query( "UPDATE {$wpdb->prefix}highest_agent_referrals SET allow_referral  = 1 WHERE allow_referral  IS NULL" );
            $wpdb->query( "UPDATE {$wpdb->prefix}highest_agent_referrals SET allow_wholesale = 1 WHERE allow_wholesale IS NULL" );
            update_option( 'hdp_status_migration_done', '1', false );
        }
    }

    public function add_cron_interval( $schedules ) {
        $schedules['thirty_minutes'] = array( 'interval' => 1800, 'display' => 'Every 30 Minutes' );
        $schedules['five_minutes']   = array( 'interval' => 300,  'display' => 'Every 5 Minutes' );
        $schedules['every_minute']   = array( 'interval' => 60,   'display' => 'Every Minute' );
        return $schedules;
    }

    // ─── Helper functions ───────────────────────────────────
    private function send_customer_receipt( $email, $phone, $package, $amount, $ref ) {
        if ( empty( $email ) || ! is_email( $email ) ) return;
        $subject = 'Your Data Purchase Confirmation - HIGHEST DATA PLUG';
        $message = "<html><body>
            <h2>Thank you for your purchase!</h2>
            <p><strong>Package:</strong> {$package}</p>
            <p><strong>Phone:</strong> {$phone}</p>
            <p><strong>Amount:</strong> GH₵" . number_format( $amount, 2 ) . "</p>
            <p><strong>Reference:</strong> {$ref}</p>
            <p>Track your order live: <a href='" . home_url( '/' ) . "'>Visit our tracker</a></p>
            <br><p>— HIGHEST DATA PLUG Team</p>
        </body></html>";
        $headers = array( 'Content-Type: text/html; charset=UTF-8' );
        wp_mail( $email, $subject, $message, $headers );
    }

    private function get_network_name( $network_id ) {
        switch ( (int) $network_id ) {
            case 3:  return 'MTN';
            case 2:  return 'Telecel';
            case 1:  return 'ATiShare';
            case 4:  return 'ATBigTime';
            case 5:  return 'MTN AFA';
            default: return 'Data';
        }
    }

    private function get_package_key( $network_id, $shared_bundle ) {
        return $this->get_network_name( $network_id ) . '-' . (int) $shared_bundle . 'GB';
    }

    public function update_order_status_by_ref( $ref, $status, $extra_data = array() ) {
        global $wpdb;
        $table = $wpdb->prefix . 'highest_orders';
        $current = $wpdb->get_var( $wpdb->prepare( "SELECT status FROM $table WHERE ref = %s", $ref ) );
        $matched_ref = $ref;

        if ( ! $current ) {
            $current = $wpdb->get_var( $wpdb->prepare( "SELECT status FROM $table WHERE incoming_api_ref = %s", $ref ) );
            if ( $current ) {
                $matched_ref = $ref;
            }
        }

        if ( ! $current ) {
            $match = $wpdb->get_var( $wpdb->prepare(
                "SELECT ref FROM $table WHERE %s LIKE CONCAT('%%', ref) LIMIT 1",
                $ref
            ) );
            if ( $match ) {
                $matched_ref = $match;
                $current = $wpdb->get_var( $wpdb->prepare( "SELECT status FROM $table WHERE ref = %s", $matched_ref ) );
                $wpdb->update( $table, array( 'incoming_api_ref' => $ref ), array( 'ref' => $matched_ref ), array( '%s' ), array( '%s' ) );
            }
        }

        if ( ! $current ) return;

        $update = array_merge( array( 'status' => $status, 'updated_at' => current_time( 'mysql' ) ), $extra_data );
        $format = array_fill( 0, count( $update ), '%s' );
        $wpdb->update( $table, $update, array( 'ref' => $matched_ref ), $format, array( '%s' ) );

        if ( $this->is_sms_enabled() && strtoupper( $status ) === 'DELIVERED' && strtoupper( $current ) !== 'DELIVERED' ) {
            $order = $wpdb->get_row( $wpdb->prepare( "SELECT * FROM $table WHERE ref = %s", $matched_ref ) );
            if ( $order ) {
                if ( $order->user_id && ! $this->is_agent_sms_disabled( $order->user_id ) && get_option( 'highest_sms_enabled_delivery_agent', 'yes' ) !== 'no' ) {
                    $agent_phone = $this->get_agent_phone( $order->user_id );
                    if ( $agent_phone ) {
                        $this->send_templated_sms( $agent_phone, 'delivery_agent', array(
                            'package' => $order->package,
                            'phone'   => $order->phone,
                        ) );
                    }
                }
                if ( ! empty( $order->phone ) ) {
                    if ( get_option( 'highest_sms_delivery_customer', 'yes' ) === 'yes' ) {
                        $this->send_templated_sms( $order->phone, 'delivery_customer', array(
                            'package' => $order->package,
                        ) );
                    }
                }
            }
        }
    }

    private function get_default_agent_prices() {
        return array(
            'MTN-1GB'   => 4.45,  'MTN-2GB'   => 9.10,  'MTN-3GB'   => 13.50, 'MTN-4GB'   => 18.10,
            'MTN-5GB'   => 22.20, 'MTN-6GB'   => 27.00, 'MTN-10GB'  => 42.00, 'MTN-15GB'  => 62.30,
            'MTN-20GB'  => 83.00, 'MTN-25GB'  => 102.00,'MTN-30GB'  => 122.00,
            'MTN-40GB'  => 160.00,'MTN-50GB'  => 199.00,'MTN-100GB' => 393.00,
            'Telecel-5GB'  => 22.00, 'Telecel-10GB' => 42.00, 'Telecel-15GB' => 59.00,
            'Telecel-20GB' => 77.00, 'Telecel-30GB' => 114.00,
            'ATiShare-1GB'  => 4.50,  'ATiShare-2GB'  => 9.00,  'ATiShare-3GB'  => 13.50,
            'ATiShare-4GB'  => 18.00, 'ATiShare-5GB'  => 22.00, 'ATiShare-6GB'  => 27.00,
            'ATiShare-7GB'  => 31.00, 'ATiShare-8GB'  => 34.50, 'ATiShare-9GB'  => 38.00,
            'ATiShare-10GB' => 42.00,
            'ATBigTime-15GB'  => 58.00, 'ATBigTime-20GB'  => 70.00,  'ATBigTime-30GB'  => 80.00,
            'ATBigTime-40GB'  => 90.00, 'ATBigTime-50GB'  => 116.00, 'ATBigTime-60GB'  => 147.00,
            'ATBigTime-80GB'  => 183.00,'ATBigTime-100GB' => 199.00, 'ATBigTime-200GB' => 385.00,
        );
    }

    private function get_agent_cost( $agent_id, $package_key ) {
        $tier = get_user_meta( $agent_id, 'highest_agent_tier', true ) ?: 'standard';
        $tier_prices = get_option( 'highest_tier_prices', array() );
        $default_prices = $this->get_default_agent_prices();
        if ( isset( $tier_prices[ $tier ][ $package_key ] ) ) {
            return $tier_prices[ $tier ][ $package_key ];
        }
        return $default_prices[ $package_key ] ?? 0;
    }

    private function get_subagent_wholesale_price( $subagent_id, $package_key ) {
        if ( get_option( 'highest_enable_wholesale_pricing', 'no' ) !== 'yes' ) return false;
        global $wpdb;
        // Also check allow_wholesale = 1 (COALESCE handles legacy NULL rows → treated as 1/allowed)
        $parent = $wpdb->get_var( $wpdb->prepare(
            "SELECT parent_agent_id FROM {$wpdb->prefix}highest_agent_referrals
             WHERE child_agent_id = %d
               AND ( status = 'active' OR status IS NULL OR status = '' )
               AND COALESCE( allow_wholesale, 1 ) = 1",
            $subagent_id
        ) );
        if ( ! $parent ) return false;
        $parent_prices = get_user_meta( $parent, 'highest_wholesale_prices', true ) ?: array();
        if ( isset( $parent_prices[ $package_key ] ) && $parent_prices[ $package_key ] > 0 ) {
            return array(
                'parent_id' => $parent,
                'price'     => $parent_prices[ $package_key ]
            );
        }
        return false;
    }

    public function activate_plugin() {
        global $wpdb;
        $charset_collate = $wpdb->get_charset_collate();
        require_once( ABSPATH . 'wp-admin/includes/upgrade.php' );

        $table_orders = $wpdb->prefix . 'highest_orders';
        $sql_orders = "CREATE TABLE IF NOT EXISTS $table_orders (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            order_type VARCHAR(10) NOT NULL DEFAULT 'walkin',
            user_id BIGINT UNSIGNED NULL,
            phone VARCHAR(20) NOT NULL,
            package VARCHAR(100) NOT NULL,
            selling_price DECIMAL(10,2) DEFAULT 0.00,
            agent_cost DECIMAL(10,2) DEFAULT 0.00,
            status VARCHAR(50) DEFAULT 'PROCESSING',
            ref VARCHAR(100) NOT NULL,
            incoming_api_ref VARCHAR(100) NULL,
            api_response LONGTEXT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            KEY idx_phone (phone),
            KEY idx_ref (ref),
            KEY idx_user (user_id),
            KEY idx_status (status)
        ) $charset_collate;";
        dbDelta( $sql_orders );

        $table_topup = $wpdb->prefix . 'highest_topup_history';
        $sql_topup = "CREATE TABLE IF NOT EXISTS $table_topup (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            user_id BIGINT UNSIGNED NOT NULL,
            amount DECIMAL(10,2) NOT NULL,
            type VARCHAR(50) NOT NULL,
            status VARCHAR(50) DEFAULT 'approved',
            reference VARCHAR(100) NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            KEY idx_user (user_id)
        ) $charset_collate;";
        dbDelta( $sql_topup );

        $table_profit = $wpdb->prefix . 'highest_profit_history';
        $sql_profit = "CREATE TABLE IF NOT EXISTS $table_profit (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            user_id BIGINT UNSIGNED NOT NULL,
            amount DECIMAL(10,2) NOT NULL,
            type VARCHAR(50) NOT NULL,
            reference VARCHAR(100) NULL,
            description TEXT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            KEY idx_user (user_id),
            KEY idx_type (type)
        ) $charset_collate;";
        dbDelta( $sql_profit );

        $table_sessions = $wpdb->prefix . 'highest_payment_sessions';
        $sql_sessions = "CREATE TABLE IF NOT EXISTS $table_sessions (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            paystack_ref VARCHAR(100) NOT NULL,
            amount DECIMAL(10,2) NOT NULL,
            phone VARCHAR(20) NOT NULL,
            email VARCHAR(100) NULL,
            package_name VARCHAR(100) NULL,
            network_id INT DEFAULT 3,
            shared_bundle INT DEFAULT 1,
            selling_price DECIMAL(10,2) DEFAULT 0,
            agent_id BIGINT UNSIGNED NULL,
            order_type VARCHAR(20) DEFAULT 'walkin',
            status VARCHAR(50) DEFAULT 'initiated',
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            KEY idx_ref (paystack_ref),
            KEY idx_status (status)
        ) $charset_collate;";
        dbDelta( $sql_sessions );

        $table_afa = $wpdb->prefix . 'highest_afa_registrations';
        $sql_afa = "CREATE TABLE IF NOT EXISTS $table_afa (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            agent_id BIGINT UNSIGNED NOT NULL,
            customer_phone VARCHAR(20) NOT NULL,
            full_name VARCHAR(255) NOT NULL,
            id_type VARCHAR(100) NOT NULL,
            id_number VARCHAR(100) NOT NULL,
            location VARCHAR(255) NOT NULL,
            dob DATE NOT NULL,
            occupation VARCHAR(255) NOT NULL,
            payment_method VARCHAR(50) NOT NULL DEFAULT 'wallet',
            paystack_ref VARCHAR(100) NULL,
            wallet_deducted TINYINT(1) DEFAULT 0,
            status VARCHAR(50) DEFAULT 'pending',
            submitted_at DATETIME NULL,
            api_response LONGTEXT NULL,
            notes TEXT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            KEY idx_agent (agent_id),
            KEY idx_status (status)
        ) $charset_collate;";
        dbDelta( $sql_afa );

        if ( get_option( 'highest_sms_enabled' ) === false ) {
            add_option( 'highest_sms_enabled', 'yes' );
        }

        // Scheduled (delayed) orders table
        $table_delayed = $wpdb->prefix . 'highest_delayed_orders';
        $sql_delayed = "CREATE TABLE IF NOT EXISTS $table_delayed (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            order_id BIGINT UNSIGNED NOT NULL,
            scheduled_at DATETIME NOT NULL,
            processed TINYINT(1) DEFAULT 0,
            KEY idx_scheduled (scheduled_at)
        ) $charset_collate;";
        dbDelta( $sql_delayed );

        $table_scheduled_sms = $wpdb->prefix . 'highest_scheduled_sms';
        $sql_scheduled_sms = "CREATE TABLE IF NOT EXISTS $table_scheduled_sms (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            recipient_type VARCHAR(20) NOT NULL DEFAULT 'agents',
            recipients TEXT NOT NULL,
            message TEXT NOT NULL,
            sender_id VARCHAR(20) DEFAULT '',
            send_at DATETIME NOT NULL,
            status VARCHAR(20) DEFAULT 'pending',
            sent_at DATETIME NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            KEY idx_status (status),
            KEY idx_send_at (send_at)
        ) $charset_collate;";
        dbDelta( $sql_scheduled_sms );

        // Agent referral relationships table
        $table_referrals = $wpdb->prefix . 'highest_agent_referrals';
        $sql_referrals = "CREATE TABLE IF NOT EXISTS $table_referrals (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            parent_agent_id BIGINT UNSIGNED NOT NULL,
            child_agent_id BIGINT UNSIGNED NOT NULL,
            bonus_type VARCHAR(10) NOT NULL DEFAULT 'percentage',
            bonus_value DECIMAL(10,2) NOT NULL DEFAULT 0,
            status VARCHAR(20) NOT NULL DEFAULT 'active',
            allow_upgrade TINYINT(1) NOT NULL DEFAULT 1,
            allow_referral TINYINT(1) NOT NULL DEFAULT 1,
            allow_wholesale TINYINT(1) NOT NULL DEFAULT 1,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            UNIQUE KEY uniq_child (child_agent_id),
            KEY idx_parent (parent_agent_id)
        ) $charset_collate;";
        dbDelta( $sql_referrals );

        // Referral bonus maturation – locked pot table
        $table_locked = $wpdb->prefix . 'highest_referral_locked';
        $sql_locked = "CREATE TABLE IF NOT EXISTS $table_locked (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            parent_agent_id BIGINT UNSIGNED NOT NULL,
            child_agent_id BIGINT UNSIGNED NOT NULL,
            order_id BIGINT UNSIGNED NULL,
            bonus_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
            status VARCHAR(20) NOT NULL DEFAULT 'locked',
            claimed_at DATETIME NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            KEY idx_parent (parent_agent_id),
            KEY idx_status (status)
        ) $charset_collate;";
        dbDelta( $sql_locked );

        // Performance indexes – safe to run on every activation (MySQL ignores duplicate index names)
        $wpdb->query( "ALTER TABLE {$wpdb->prefix}highest_orders ADD INDEX IF NOT EXISTS idx_user_status (user_id, status)" );
        $wpdb->query( "ALTER TABLE {$wpdb->prefix}highest_agent_referrals ADD INDEX IF NOT EXISTS idx_parent_status (parent_agent_id, status)" );

        // v5.3: Add per-sub-agent feature permission columns (safe to run on every activation)
        $wpdb->query( "ALTER TABLE {$wpdb->prefix}highest_agent_referrals
            ADD COLUMN IF NOT EXISTS allow_upgrade   TINYINT(1) DEFAULT 1,
            ADD COLUMN IF NOT EXISTS allow_referral  TINYINT(1) DEFAULT 1,
            ADD COLUMN IF NOT EXISTS allow_wholesale TINYINT(1) DEFAULT 1" );

        // Backfill NULL permission values for rows inserted before v5.3
        // (ALTER TABLE ADD COLUMN does not populate existing rows with the DEFAULT value)
        $wpdb->query( "UPDATE {$wpdb->prefix}highest_agent_referrals
            SET allow_upgrade   = 1 WHERE allow_upgrade   IS NULL" );
        $wpdb->query( "UPDATE {$wpdb->prefix}highest_agent_referrals
            SET allow_referral  = 1 WHERE allow_referral  IS NULL" );
        $wpdb->query( "UPDATE {$wpdb->prefix}highest_agent_referrals
            SET allow_wholesale = 1 WHERE allow_wholesale IS NULL" );

        // Ensure the status column has a proper DEFAULT and fix any existing NULL/empty rows
        // (rows inserted before 'status' was explicitly set in the INSERT will be NULL)
        $wpdb->query( "ALTER TABLE {$wpdb->prefix}highest_agent_referrals
            MODIFY COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active'" );
        $wpdb->query( "UPDATE {$wpdb->prefix}highest_agent_referrals
            SET status = 'active'
            WHERE status IS NULL OR status = ''" );

        // WAEC Vouchers table
        $table_waec_vouchers = $wpdb->prefix . 'highest_waec_vouchers';
        $sql_waec_vouchers = "CREATE TABLE IF NOT EXISTS $table_waec_vouchers (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            exam_type VARCHAR(30) NOT NULL DEFAULT 'WASSCE',
            serial_number VARCHAR(30) NOT NULL,
            pin VARCHAR(30) NOT NULL,
            status VARCHAR(20) NOT NULL DEFAULT 'available',
            used_order_id BIGINT UNSIGNED NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            KEY idx_status (status),
            UNIQUE KEY idx_serial (serial_number)
        ) $charset_collate;";
        dbDelta( $sql_waec_vouchers );

        // WAEC Orders table
        $table_waec_orders = $wpdb->prefix . 'highest_waec_orders';
        $sql_waec_orders = "CREATE TABLE IF NOT EXISTS $table_waec_orders (
            id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
            customer_email VARCHAR(100) NOT NULL,
            customer_phone VARCHAR(20) NOT NULL,
            exam_type VARCHAR(30) NOT NULL,
            paystack_ref VARCHAR(100) NULL,
            amount DECIMAL(10,2) NOT NULL DEFAULT 0,
            voucher_id BIGINT UNSIGNED NULL,
            status VARCHAR(20) NOT NULL DEFAULT 'pending_payment',
            pin_sent_at DATETIME NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            KEY idx_phone (customer_phone),
            KEY idx_email (customer_email),
            KEY idx_status (status)
        ) $charset_collate;";
        dbDelta( $sql_waec_orders );
    }

    public function enqueue_assets() {
    wp_enqueue_style( 'font-awesome', 'https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css' );
    wp_enqueue_script( 'tailwind', 'https://cdn.tailwindcss.com', array(), null, true );
    wp_enqueue_script( 'chartjs', 'https://cdn.jsdelivr.net/npm/chart.js', array(), null, true );
    wp_enqueue_script( 'paystack', 'https://js.paystack.co/v1/inline.js', array(), null, true );
    wp_add_inline_script( 'tailwind', '
        const HIGHEST_AJAX = "' . admin_url( "admin-ajax.php" ) . '";
        const HIGHEST_NONCE = "' . wp_create_nonce( "highest_data_plug" ) . '";
        const PAYSTACK_PUBLIC_KEY = "' . esc_js( get_option( "highest_paystack_public_key", "" ) ) . '";
        const AGENT_PORTAL_URL = "' . esc_js( home_url( "/agent-portal/" ) ) . '";
        const HOME_URL = "' . esc_js( home_url() ) . '";
    ' );

    // Decide which PWA manifest to use
    if ( is_page( 'agent-portal' ) ) {
        // Agent App
        echo '<link rel="manifest" href="' . plugin_dir_url( __FILE__ ) . 'manifest.json">';
        
        // Service Worker for Agent Portal
        wp_add_inline_script( 'tailwind', "
            if ('serviceWorker' in navigator) {
                navigator.serviceWorker.register('" . plugin_dir_url( __FILE__ ) . "sw.js')
                    .then(reg => console.log('Agent SW registered'))
                    .catch(err => console.log('SW registration failed', err));
            }
        " );
    } else {
        // Public / Walk‑in Customer App
        echo '<link rel="manifest" href="' . plugin_dir_url( __FILE__ ) . 'manifest-public.json">';
        
        // Service Worker for Public Buy
        wp_add_inline_script( 'tailwind', "
            if ('serviceWorker' in navigator) {
                navigator.serviceWorker.register('" . plugin_dir_url( __FILE__ ) . "sw-public.js')
                    .then(reg => console.log('Public SW registered'))
                    .catch(err => console.log('SW registration failed', err));
            }
        " );
    }
}
        
    private function get_dakazina_api_key() { return get_option( 'highest_dakazina_api_key', '' ); }
    private function is_sms_enabled() { return get_option( 'highest_sms_enabled', 'yes' ) === 'yes'; }
    private function is_agent_sms_disabled( $user_id ) { return (bool) get_user_meta( $user_id, 'highest_sms_disabled', true ); }
    private function get_agent_phone( $user_id ) {
        $phone = get_user_meta( $user_id, 'phone', true );
        if ( empty( $phone ) ) $phone = get_user_meta( $user_id, 'registration_phone', true );
        return $phone;
    }

    /**
     * Normalise a Ghanaian phone number to the 10-digit local format 0551234567.
     * Accepts 9, 10, 12 or 13 digit inputs with or without spaces / plus.
     */
    private function normalise_phone( $phone ) {
        $digits = preg_replace( '/\D/', '', trim( $phone ) );
        if ( strlen( $digits ) === 9 ) {
            return '0' . $digits;
        }
        if ( strlen( $digits ) === 12 && substr( $digits, 0, 3 ) === '233' ) {
            return '0' . substr( $digits, 3, 9 );
        }
        if ( strlen( $digits ) === 13 && substr( $digits, 0, 4 ) === '+233' ) {
            return '0' . substr( $digits, 4, 9 );
        }
        if ( strlen( $digits ) === 10 && $digits[0] === '0' ) {
            return $digits;
        }
        // fallback – keep as-is
        return $digits;
    }

    private function dakazina_api_call( $endpoint, $method = 'GET', $body = null ) {
        $api_key = $this->get_dakazina_api_key();
        if ( empty( $api_key ) ) return array( 'error' => 'DataKazina API key not configured.' );
        $url = 'https://reseller.dakazinabusinessconsult.com/api/v1/' . ltrim( $endpoint, '/' );
        $args = array(
            'headers' => array(
                'x-api-key'    => $api_key,
                'Content-Type' => 'application/json',
                'Accept'       => 'application/json',
            ),
            'method'  => $method,
            'timeout' => 30,
        );
        if ( $body !== null ) $args['body'] = json_encode( $body );
        $response = wp_remote_request( $url, $args );
        if ( is_wp_error( $response ) ) return array( 'error' => $response->get_error_message() );
        $http_code = wp_remote_retrieve_response_code( $response );
        $raw_body  = wp_remote_retrieve_body( $response );
        $result    = json_decode( $raw_body, true );
        return array( 'data' => $result, 'http_code' => $http_code, 'raw_body' => $raw_body );
    }

    private function hubtel_send_sms( $to, $message ) {
        if ( ! $this->is_sms_enabled() ) return 'SMS globally disabled.';
        $client_id = get_option( 'highest_hubtel_client_id', '' );
        $client_secret = get_option( 'highest_hubtel_client_secret', '' );
        $sender = get_option( 'highest_hubtel_sender_id', 'HIGHEST' );
        if ( empty( $client_id ) || empty( $client_secret ) ) return 'Hubtel credentials missing.';
        $to = preg_replace( '/\D/', '', $to );
        if ( strlen( $to ) === 9 ) $to = '233' . $to;
        if ( strlen( $to ) === 10 ) $to = '233' . substr( $to, -9 );
        $url = 'https://smsc.hubtel.com/v1/messages/send';
        $url = add_query_arg( array(
            'clientid'     => $client_id,
            'clientsecret' => $client_secret,
            'from'         => $sender,
            'to'           => $to,
            'content'      => $message,
        ), $url );
        $response = wp_remote_get( $url, array( 'timeout' => 15 ) );
        if ( is_wp_error( $response ) ) return $response->get_error_message();
        $status_code = wp_remote_retrieve_response_code( $response );
        $resp_body = json_decode( wp_remote_retrieve_body( $response ), true );
        if ( $status_code === 200 && isset( $resp_body['status'] ) && $resp_body['status'] === 0 ) return true;
        $error_msg = $resp_body['statusDescription'] ?? $resp_body['message'] ?? 'Unknown error';
        error_log( "Hubtel SMS error: $error_msg" );
        return "Hubtel SMS failed: $error_msg";
    }

    /**
     * Send SMS via Nalo Solutions API.
     * Handles plain-text response (1701|... = success).
     */
    private function nalo_send_sms( $to, $message ) {
        $api_key = get_option( 'highest_nalo_api_key', '' );
        $sender  = get_option( 'highest_nalo_sender_id', '' );
        if ( empty( $api_key ) || empty( $sender ) ) {
            return 'Nalo Solutions credentials missing.';
        }

        // Normalise phone to 233XXXXXXXXX (12 digits, no +)
        $digits = preg_replace( '/\D/', '', trim( $to ) );
        if ( strlen( $digits ) === 9 ) {
            $to = '233' . $digits;
        } elseif ( strlen( $digits ) === 10 && $digits[0] === '0' ) {
            $to = '233' . substr( $digits, 1 );
        } elseif ( strlen( $digits ) === 12 && substr( $digits, 0, 3 ) === '233' ) {
            $to = $digits;
        } elseif ( strlen( $digits ) === 13 && substr( $digits, 0, 4 ) === '+233' ) {
            $to = substr( $digits, 1 );
        }

        $url = add_query_arg( array(
            'key'         => $api_key,
            'type'        => '0',
            'destination' => $to,
            'source'      => $sender,
            'message'     => $message,
            'dlr'         => '1',
        ), 'https://sms.nalosolutions.com/smsbackend/clientapi/Resl_Nalo/send-message/' );

        $response = wp_remote_get( $url, array( 'timeout' => 45 ) );
        if ( is_wp_error( $response ) ) {
            return $response->get_error_message();
        }

        $body = trim( wp_remote_retrieve_body( $response ) );

        // Success: response starts with "1701|"
        if ( strpos( $body, '1701|' ) === 0 ) {
            return true;
        }

        // Error codes
        $error_codes = array(
            '1702' => 'Invalid URL / missing parameter',
            '1703' => 'Invalid username / password / API key',
            '1704' => 'Invalid type field',
            '1705' => 'Invalid message',
            '1706' => 'Invalid destination phone number',
            '1707' => 'Invalid sender ID',
            '1708' => 'Invalid DLR value',
            '1709' => 'User validation failed',
            '1710' => 'Internal error',
            '1025' => 'Insufficient user credit',
            '1026' => 'Insufficient reseller credit',
        );
        $error_msg = $error_codes[ $body ] ?? "Nalo API error (code: {$body})";
        error_log( "Nalo SMS error: {$error_msg}" );
        return "Nalo SMS failed: {$error_msg}";
    }

    /**
     * Send SMS via Moolre (GET endpoint).
     * Docs: https://docs.moolre.com/#/send-sms-get
     */
    private function moolre_send_sms( $to, $message ) {
        $vas_key = get_option( 'highest_moolre_vas_key', '' );
        $sender  = get_option( 'highest_moolre_sender_id', '' );

        if ( empty( $vas_key ) || empty( $sender ) ) {
            return 'Moolre SMS credentials missing.';
        }

        // Normalise to 233XXXXXXXXX (international, no +)
        $digits = preg_replace( '/\D/', '', trim( $to ) );
        if ( strlen( $digits ) === 9 ) {
            $to = '233' . $digits;
        } elseif ( strlen( $digits ) === 10 && $digits[0] === '0' ) {
            $to = '233' . substr( $digits, 1 );
        } elseif ( strlen( $digits ) === 12 && substr( $digits, 0, 3 ) === '233' ) {
            $to = $digits;
        } elseif ( strlen( $digits ) === 13 && substr( $digits, 0, 4 ) === '+233' ) {
            $to = substr( $digits, 1 );
        }

        $url = add_query_arg( array(
            'type'         => '1',
            'senderid'     => $sender,
            'recipient'    => $to,
            'message'      => $message,
            'X-API-VASKEY' => $vas_key,
        ), 'https://api.moolre.com/open/sms/send' );

        $response = wp_remote_get( $url, array( 'timeout' => 30 ) );
        if ( is_wp_error( $response ) ) {
            return $response->get_error_message();
        }

        $body = json_decode( wp_remote_retrieve_body( $response ), true );
        if ( ! empty( $body['status'] ) && $body['status'] == 1 ) {
            return true;
        }

        $error_msg = $body['message'] ?? ( $body['code'] ?? 'Unknown Moolre SMS error' );
        error_log( "Moolre SMS error: {$error_msg}" );
        return "Moolre SMS failed: {$error_msg}";
    }

    /**
     * Initiate a payment prompt.
     * - For walk‑in/agent (web): calls Moolre Generate Payment Link API, returns a URL.
     * - For USSD (offline): calls Moolre Initiate Payment API, pushes MoMo prompt directly.
     * - For Paystack: returns true (inline popup ready).
     *
     * @param string $phone       Payer phone number (local or international).
     * @param float  $amount      Amount in GHS.
     * @param string $reference   Unique external reference.
     * @param string $description Description for the payment.
     * @param string $email       Payer email (used for web flows).
     * @param string $channel     USSD only – 'mtn', 'telecel', or 'at'.
     * @return true|string|WP_Error  true for Paystack, URL for Moolre web, WP_Error on failure.
     */
    /**
     * Initiate a payment prompt.
     *
     * @param string $phone       Payer phone number (local or international).
     * @param float  $amount      Amount in GHS.
     * @param string $reference   Unique external reference.
     * @param string $description Description for the payment.
     * @param string $email       Payer email (used for web flows).
     * @param string $channel     USSD only – 'mtn', 'telecel', or 'at'.
     * @param string $gateway     Optional. Override the global gateway. Empty = use global.
     * @return true|string|WP_Error  true for Paystack, URL for Moolre web, WP_Error on failure.
     */
    private function initiate_payment_prompt( $phone, $amount, $reference, $description = '', $email = '', $channel = '', $gateway = '' ) {
        if ( empty( $gateway ) ) {
            $gateway = get_option( 'highest_payment_gateway', 'paystack' );
        }

        if ( $gateway !== 'moolre' ) {
            // Paystack – inline popup is ready
            return true;
        }

        // ── Moolre Gateway ──────────────────────────────
        $api_user = get_option( 'highest_moolre_api_user', '' );
        $pub_key  = get_option( 'highest_moolre_pub_key', '' );
        $acct_num = get_option( 'highest_moolre_account_number', '' );

        if ( empty( $api_user ) || empty( $pub_key ) || empty( $acct_num ) ) {
            return new WP_Error( 'moolre_credentials', 'Moolre credentials missing.' );
        }

        // Normalise phone to 233XXXXXXXXX
        $digits = preg_replace( '/\D/', '', trim( $phone ) );
        if ( strlen( $digits ) === 9 )      $phone = '233' . $digits;
        elseif ( strlen( $digits ) === 10 && $digits[0] === '0' ) $phone = '233' . substr( $digits, 1 );

        // If a channel is provided, this is a USSD offline MoMo prompt
        if ( ! empty( $channel ) ) {
            $channel_map = array(
                'mtn'      => '13',
                'telecel'  => '6',
                'at'       => '7',
            );
            $moolre_channel = $channel_map[ $channel ] ?? '13';

            $response = wp_remote_post( 'https://api.moolre.com/open/transact/payment', array(
                'headers' => array(
                    'X-API-USER'   => $api_user,
                    'X-API-PUBKEY' => $pub_key,
                    'Content-Type' => 'application/json',
                ),
                'body' => json_encode( array(
                    'type'          => 1,
                    'channel'       => $moolre_channel,
                    'currency'      => 'GHS',
                    'payer'         => $phone,
                    'amount'        => $amount,
                    'externalref'   => $reference,
                    'accountnumber' => $acct_num,
                ) ),
                'timeout' => 30,
            ) );

            if ( is_wp_error( $response ) ) return $response;

            $body = json_decode( wp_remote_retrieve_body( $response ), true );
            $code = $body['code'] ?? '';

            // TP14 = OTP sent, TP11 = prompt sent
            if ( in_array( $code, array( 'TP11', 'TP14' ) ) ) {
                return true; // MoMo prompt pushed
            }

            $error_msg = $body['message'] ?? "Moolre payment failed (code: {$code})";
            error_log( "Moolre USSD payment error: {$error_msg}" );
            return new WP_Error( 'moolre_payment_failed', $error_msg );
        }

        // ── Web flow: Generate Payment Link ─────────────
        $response = wp_remote_post( 'https://api.moolre.com/embed/link', array(
            'headers' => array(
                'X-API-USER'   => $api_user,
                'X-API-PUBKEY' => $pub_key,
                'Content-Type' => 'application/json',
            ),
            'body' => json_encode( array(
                'type'          => 1,
                'amount'        => $amount,
                'email'         => $email ?: $phone . '@highestdataplug.com',
                'externalref'   => $reference,
                'callback'      => home_url( '/wp-json/highest/v1/moolre-webhook' ),
                'redirect'      => home_url( '/?payment=success&ref=' . $reference ),
                'reusable'      => '0',
                'currency'      => 'GHS',
                'accountnumber' => $acct_num,
            ) ),
            'timeout' => 30,
        ) );

        if ( is_wp_error( $response ) ) return $response;

        $body = json_decode( wp_remote_retrieve_body( $response ), true );

        if ( ! empty( $body['status'] ) && $body['status'] == 1 && ! empty( $body['data']['authorization_url'] ) ) {
            return $body['data']['authorization_url'];
        }

        $error_msg = $body['message'] ?? 'Unknown Moolre link generation error';
        error_log( "Moolre link generation error: {$error_msg}" );
        return new WP_Error( 'moolre_link_failed', $error_msg );
    }

    /**
     * AJAX: generate a Moolre checkout URL for the walk-in / agent JS.
     * Called when highest_payment_gateway === 'moolre'.
     */
    public function ajax_get_moolre_checkout_url() {
        $this->verify_ajax_request();
        $ref     = sanitize_text_field( $_POST['ref'] ?? '' );
        $amount  = floatval( $_POST['amount'] ?? 0 );
        $phone   = sanitize_text_field( $_POST['phone'] ?? '' );
        $email   = sanitize_email( $_POST['email'] ?? '' );
        $desc    = sanitize_text_field( $_POST['description'] ?? '' );

        $result = $this->initiate_payment_prompt( $phone, $amount, $ref, $desc, $email );
        if ( is_wp_error( $result ) ) {
            wp_send_json_error( $result->get_error_message() );
        }
        if ( $result === true ) {
            wp_send_json_error( 'Paystack is active – use inline popup.' );
        }
        wp_send_json_success( array( 'url' => $result ) );
    }

    /**
     * Send an SMS to a phone number using the configured provider for the given message type.
     *
     * @param string $to           Recipient phone number (any supported format).
     * @param string $message      The SMS body.
     * @param string $message_type The message-type key (used to look up provider and on/off toggle).
     * @return true|string         Returns true on success, or an error string on failure.
     */
    private function send_sms( $to, $message, $message_type = 'order_received' ) {
        if ( ! $this->is_sms_enabled() ) return 'SMS globally disabled.';
        if ( get_option( "highest_sms_enabled_{$message_type}", 'yes' ) === 'no' ) {
            return "SMS type {$message_type} disabled.";
        }
        $provider = get_option( "highest_sms_provider_{$message_type}", 'hubtel' );
        if ( $provider === 'bulksmsgh' ) {
            return $this->bulksmsgh_send_sms( $to, $message );
        } elseif ( $provider === 'nalo' ) {
            return $this->nalo_send_sms( $to, $message );
        } elseif ( $provider === 'moolre' ) {
            return $this->moolre_send_sms( $to, $message );
        }
        return $this->hubtel_send_sms( $to, $message );
    }

    /**
     * Send an SMS using an admin-editable template stored in the DB.
     * Falls back to a sensible hard-coded default if no template has been saved yet.
     * Replacements array: [ 'key' => 'value', ... ] maps to {key} placeholders.
     */
    private function send_templated_sms( $to, $message_type, $replacements = array() ) {
        static $hardcoded_defaults = null;
        if ( $hardcoded_defaults === null ) {
            $hardcoded_defaults = array(
                'order_received'     => 'We have received your order for {package}. Your data will be delivered shortly. Track at: {tracking_url}',
                'delivery_customer'  => 'Your data package ({package}) has been delivered successfully. Kindly allow sometime to reflect. Thank you for choosing.',
                'delivery_agent'     => 'Order {package} to {phone} has been DELIVERED. Thank you.',
                'order_placed_agent' => 'Order PLACED: {package} to {phone} has been submitted and is being processed.',
                'wallet_topup'       => 'Your wallet has been credited with GH\xe2\x82\xb5{amount}. New balance: GH\xe2\x82\xb5{balance}',
                'withdrawal_update'  => 'Your withdrawal request of GH\xe2\x82\xb5{amount} has been approved. Funds will be sent to your account shortly.',
                'manual_topup_agent' => 'Your manual top-up of GH\xe2\x82\xb5{amount} has been approved. New wallet balance: GH\xe2\x82\xb5{balance}',
                'welcome_agent'      => 'Welcome {name}! Your HIGHEST DATA PLUG agent account is ready. Login at {login_url}',
                'emergency_admin'    => 'EMERGENCY: Payment verified (ref: {ref}) but data delivery FAILED. Package: {package} to {phone}. Amount: GH\xe2\x82\xb5{amount}. Agent ID: {agent_id}. Please check manually.',
                'retry_sms'          => 'We noticed your purchase attempt was not completed. Please try again or contact support on {support}.',
                'notify_agent'       => 'Customer {phone} attempted to buy {package} but the payment did not complete. Please assist them.',
                'scheduled_campaign' => '{message}',
                'bulk_agents'        => '{message}',
                'bulk_customers'     => '{message}',
            );
        }
        $template = get_option( "highest_sms_template_{$message_type}", '' );
        if ( empty( trim( $template ) ) ) {
            $template = $hardcoded_defaults[ $message_type ] ?? '';
        }
        if ( empty( $template ) ) {
            return 'No SMS template configured for: ' . $message_type;
        }
        $message = $template;
        foreach ( $replacements as $key => $value ) {
            $message = str_replace( '{' . $key . '}', $value, $message );
        }
        return $this->send_sms( $to, $message, $message_type );
    }

    private function bulksmsgh_send_sms( $to, $message ) {
        $api_key = get_option( 'highest_bulksmsgh_api_key', '' );
        $sender  = get_option( 'highest_bulksmsgh_sender_id', '' );
        if ( empty( $api_key ) || empty( $sender ) ) return 'BulkSMS Ghana credentials missing.';

        // BulkSMS Ghana expects 233XXXXXXXXX format (no leading 0, no +)
        $digits = preg_replace( '/\D/', '', trim( $to ) );
        if ( strlen( $digits ) === 9 ) {
            $to = '233' . $digits;
        } elseif ( strlen( $digits ) === 10 && $digits[0] === '0' ) {
            $to = '233' . substr( $digits, 1 );
        } elseif ( strlen( $digits ) === 12 && substr( $digits, 0, 3 ) === '233' ) {
            $to = $digits;
        } elseif ( strlen( $digits ) === 13 && substr( $digits, 0, 3 ) === '233' ) {
            $to = $digits;
        } else {
            $to = $digits;
        }

        $url = add_query_arg( array(
            'key'       => $api_key,
            'to'        => $to,
            'msg'       => $message,
            'sender_id' => $sender,
        ), 'https://clientlogin.bulksmsgh.com/smsapi' );

        $response = wp_remote_get( $url, array( 'timeout' => 15 ) );
        if ( is_wp_error( $response ) ) return $response->get_error_message();

        $body = wp_remote_retrieve_body( $response );
        $code = intval( trim( $body ) );

        if ( $code === 1000 ) return true;

        $errors = array(
            1002 => 'SMS sending failed',
            1003 => 'Insufficient balance',
            1004 => 'Invalid API key',
            1005 => 'Invalid phone number',
            1006 => 'Invalid sender ID',
            1007 => 'Message scheduled for later delivery',
            1008 => 'Empty message',
        );
        $error_msg = $errors[ $code ] ?? "Unknown error (code {$code})";
        error_log( "BulkSMS Ghana error: {$error_msg}" );
        return "BulkSMS Ghana failed: {$error_msg}";
    }

    private function add_profit_history( $user_id, $amount, $type, $reference = '', $description = '' ) {
        global $wpdb;
        $wpdb->insert( $wpdb->prefix . 'highest_profit_history', array(
            'user_id'     => $user_id,
            'amount'      => $amount,
            'type'        => $type,
            'reference'   => $reference,
            'description' => $description,
            'created_at'  => current_time( 'mysql' ),
        ) );
    }

    private function send_order_received_sms( $customer_phone, $package_name, $order_ref, $agent_store_url = null ) {
        if ( ! $this->is_sms_enabled() ) return;
        if ( get_option( 'highest_sms_enabled_order_received', 'yes' ) !== 'yes' ) return;
        if ( empty( $customer_phone ) ) return;
        $tracking_url = ! empty( $agent_store_url ) ? $agent_store_url : home_url( '/' );
        $this->send_templated_sms( $customer_phone, 'order_received', array(
            'package'      => $package_name,
            'tracking_url' => $tracking_url,
        ) );
    }

    // ─── Transient + Session Logging (v5.1 – emergency SMS on DK failure) ──
    public function ajax_log_payment_initiation() {
        $this->verify_ajax_request();
        $ref = sanitize_text_field( $_POST['ref'] );
        $order_data = array(
            'phone'         => sanitize_text_field( $_POST['phone'] ),
            'email'         => sanitize_email( $_POST['email'] ),
            'package_name'  => sanitize_text_field( $_POST['package_name'] ?? '' ),
            'network_id'    => intval( $_POST['network_id'] ?? 3 ),
            'shared_bundle' => intval( $_POST['shared_bundle'] ?? 1 ),
            'selling_price' => floatval( $_POST['selling_price'] ?? 0 ),
            'agent_id'      => intval( $_POST['agent_id'] ?? 0 ),
            'order_type'    => sanitize_text_field( $_POST['order_type'] ?? 'walkin' ),
        );
        set_transient( 'hdp_order_' . $ref, $order_data, 30 * MINUTE_IN_SECONDS );

        global $wpdb;
        $wpdb->insert( $wpdb->prefix . 'highest_payment_sessions', array(
            'paystack_ref'   => $ref,
            'amount'         => floatval( $_POST['amount'] ),
            'phone'          => $order_data['phone'],
            'email'          => $order_data['email'],
            'package_name'   => $order_data['package_name'],
            'network_id'     => $order_data['network_id'],
            'shared_bundle'  => $order_data['shared_bundle'],
            'selling_price'  => $order_data['selling_price'],
            'agent_id'       => $order_data['agent_id'],
            'order_type'     => $order_data['order_type'],
            'status'         => 'initiated',
            'created_at'     => current_time( 'mysql' ),
        ) );
        wp_send_json_success( array( 'session_id' => $wpdb->insert_id ) );
    }

    private function create_order_from_transient( $ref ) {
        $transient_key = 'hdp_order_' . $ref;
        $data = get_transient( $transient_key );
        if ( ! $data ) {
            global $wpdb;
            $session = $wpdb->get_row( $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}highest_payment_sessions WHERE paystack_ref = %s AND status = 'initiated'", $ref ) );
            if ( ! $session ) return false;
            $data = array(
                'phone'         => $session->phone,
                'email'         => $session->email,
                'package_name'  => $session->package_name,
                'network_id'    => $session->network_id,
                'shared_bundle' => $session->shared_bundle,
                'selling_price' => $session->selling_price,
                'agent_id'      => $session->agent_id,
                'order_type'    => $session->order_type,
            );
        } else {
            delete_transient( $transient_key );
        }

        // ── Wallet top-up handler (must run before any order logic) ──
        if ( ( $data['order_type'] ?? '' ) === 'wallet_topup' ) {
            $user_id = $data['agent_id'] ? intval( $data['agent_id'] ) : null;
            if ( ! $user_id ) return false;

            // Prevent double-credit
            $processed = get_user_meta( $user_id, 'highest_processed_topups', true ) ?: array();
            if ( in_array( $ref, $processed ) ) return true;

            $amount     = floatval( $data['selling_price'] );
            $wallet     = (float) get_user_meta( $user_id, 'highest_wallet', true );
            $new_wallet = $wallet + $amount;
            update_user_meta( $user_id, 'highest_wallet', $new_wallet );

            $processed[] = $ref;
            update_user_meta( $user_id, 'highest_processed_topups', $processed );

            global $wpdb;
            $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                'user_id'    => $user_id,
                'amount'     => $amount,
                'type'       => 'paystack',
                'status'     => 'approved',
                'reference'  => $ref,
                'created_at' => current_time( 'mysql' ),
            ) );
            $wpdb->update( $wpdb->prefix . 'highest_topup_history',
                array( 'balance_after' => $new_wallet ),
                array( 'id' => $wpdb->insert_id )
            );
            $wpdb->update( $wpdb->prefix . 'highest_payment_sessions',
                array( 'status' => 'paid_validated' ),
                array( 'paystack_ref' => $ref )
            );

            if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) ) {
                $phone = $this->get_agent_phone( $user_id );
                if ( $phone ) {
                    $this->send_templated_sms( $phone, 'wallet_topup', array(
                        'amount'  => number_format( $amount, 2 ),
                        'balance' => number_format( $new_wallet, 2 ),
                    ) );
                }
            }
            return true;
        }
        // ── End wallet top-up handler ─────────────────────────────────

        global $wpdb;
        $exists = $wpdb->get_var( $wpdb->prepare( "SELECT id FROM {$wpdb->prefix}highest_orders WHERE ref = %s", $ref ) );
        if ( $exists ) {
            $wpdb->update( $wpdb->prefix . 'highest_payment_sessions', array( 'status' => 'paid_validated' ), array( 'paystack_ref' => $ref ) );
            return true;
        }


        $agent_cost = 0;
        $agent_store_url = null;
        $subagent_parent_id = null;
        if ( $data['agent_id'] ) {
            $pkg_key = $this->get_package_key( $data['network_id'], $data['shared_bundle'] );
            $wholesale_info = $this->get_subagent_wholesale_price( $data['agent_id'], $pkg_key );
            if ( $wholesale_info ) {
                $agent_cost = $wholesale_info['price'];
                $subagent_parent_id = $wholesale_info['parent_id'];
            } else {
                $agent_cost = $this->get_agent_cost( $data['agent_id'], $pkg_key );
            }
            $store_name = get_user_meta( $data['agent_id'], 'highest_store_name', true );
            if ( $store_name ) $agent_store_url = home_url( '/store/' . sanitize_title( $store_name ) );
        }

        // Fix: AirtelTigo network detection for walk-in packages
        $network_id = $data['network_id'];
        if ( stripos( $data['package_name'], 'AirtelTigo' ) !== false ) {
            preg_match( '/(\d+)\s*GB/i', $data['package_name'], $m );
            $gb         = (int) ( $m[1] ?? 0 );
            $network_id = ( $gb >= 15 ) ? 4 : 1;
        }

        $wpdb->insert( $wpdb->prefix . 'highest_orders', array(
            'order_type'    => $data['order_type'],
            'user_id'       => $data['agent_id'] ?: null,
            'phone'         => $this->normalise_phone( $data['phone'] ),
            'package'       => $data['package_name'],
            'selling_price' => $data['selling_price'],
            'agent_cost'    => $agent_cost,
            'status'        => 'PLACED',
            'ref'           => $ref,
            'incoming_api_ref' => '',
            'api_response'  => '',
            'created_at'    => current_time( 'mysql' ),
        ) );
        $order_id = $wpdb->insert_id;

        // Attempt to send to DataKazina – if it fails, send emergency SMS (automatic delivery failure)
        $dk_success = $this->send_order_to_dakazina( $order_id );
        if ( ! $dk_success && $this->emergency_sms_enabled && ! empty( $this->emergency_sms_phone ) && get_option( 'highest_sms_emergency_admin', 'yes' ) === 'yes' ) {
            $this->send_templated_sms( $this->emergency_sms_phone, 'emergency_admin', array(
                'ref'      => $ref,
                'package'  => $data['package_name'],
                'phone'    => $data['phone'],
                'amount'   => number_format( (float) $data['selling_price'], 2 ),
                'agent_id' => $data['agent_id'],
            ) );
        }

        if ( $data['agent_id'] && $agent_cost > 0 && $data['selling_price'] > $agent_cost ) {
            $profit_earned = round( $data['selling_price'] - $agent_cost, 2 );
            update_user_meta( $data['agent_id'], 'highest_profit', (float) get_user_meta( $data['agent_id'], 'highest_profit', true ) + $profit_earned );
            $this->add_profit_history( $data['agent_id'], $profit_earned, 'profit_earned', $ref, "Sale via {$data['order_type']} to {$data['phone']}" );
        }

        // Referral bonus for parent agent (wholesale or percentage)
        if ( get_option( 'highest_enable_referral_system', 'no' ) === 'yes' && ! empty( $data['agent_id'] ) && isset( $profit_earned ) && $profit_earned > 0 ) {
            global $wpdb;
            if ( $subagent_parent_id ) {
                // Wholesale margin for parent
                $pkg_key_ref = $this->get_package_key( $data['network_id'], $data['shared_bundle'] );
                $parent_tier_cost = $this->get_agent_cost( $subagent_parent_id, $pkg_key_ref );
                $parent_profit_amount = round( $agent_cost - $parent_tier_cost, 2 );
                if ( $parent_profit_amount > 0 ) {
                    if ( get_option( 'highest_enable_referral_maturation', 'yes' ) === 'yes' ) {
                        $wpdb->insert( $wpdb->prefix . 'highest_referral_locked', array(
                            'parent_agent_id' => $subagent_parent_id,
                            'child_agent_id'  => $data['agent_id'],
                            'order_id'        => $order_id,
                            'bonus_amount'    => $parent_profit_amount,
                            'status'          => 'locked',
                            'created_at'      => current_time( 'mysql' ),
                        ) );
                    } else {
                        $parent_current = (float) get_user_meta( $subagent_parent_id, 'highest_profit', true );
                        update_user_meta( $subagent_parent_id, 'highest_profit', $parent_current + $parent_profit_amount );
                        $this->add_profit_history( $subagent_parent_id, $parent_profit_amount, 'wholesale_profit', 'ref-' . $data['agent_id'], "Wholesale profit from sub-agent sale" );
                    }
                }
            } else {
                $referral = $wpdb->get_row( $wpdb->prepare(
                    "SELECT parent_agent_id, bonus_value FROM {$wpdb->prefix}highest_agent_referrals
                     WHERE child_agent_id = %d
                       AND COALESCE( status, 'active' ) = 'active'", $data['agent_id']
                ) );
                if ( $referral ) {
                    $pct = $referral->bonus_value > 0 ? intval( $referral->bonus_value ) : intval( get_user_meta( $referral->parent_agent_id, 'highest_referral_bonus_pct', true ) );
                    $limit = intval( get_option( 'highest_referral_bonus_limit_pct', 20 ) );
                    if ( $limit > 0 && $pct > $limit ) $pct = $limit;
                    if ( $pct > 0 ) {
                        $bonus = round( $profit_earned * ( $pct / 100 ), 2 );
                        if ( get_option( 'highest_enable_referral_maturation', 'yes' ) === 'yes' ) {
                            // Lock bonus in DB table – not yet withdrawable
                            $wpdb->insert( $wpdb->prefix . 'highest_referral_locked', array(
                                'parent_agent_id' => $referral->parent_agent_id,
                                'child_agent_id'  => $data['agent_id'],
                                'order_id'        => $order_id,
                                'bonus_amount'    => $bonus,
                                'status'          => 'locked',
                                'created_at'      => current_time( 'mysql' ),
                            ) );
                        } else {
                            // Direct credit – original behaviour when maturation is disabled
                            $parent_profit = (float) get_user_meta( $referral->parent_agent_id, 'highest_profit', true );
                            update_user_meta( $referral->parent_agent_id, 'highest_profit', $parent_profit + $bonus );
                            $this->add_profit_history( $referral->parent_agent_id, $bonus, 'referral_bonus_percentage', 'ref-' . $data['agent_id'], "{$pct}% referral bonus from sub-agent sale" );
                        }
                    }
                }
            }
        }

        $this->send_order_received_sms( $data['phone'], $data['package_name'], $ref, $agent_store_url );
        if ( ! empty( $data['email'] ) ) {
            $this->send_customer_receipt( $data['email'], $data['phone'], $data['package_name'], $data['selling_price'], $ref );
        }
        $wpdb->update( $wpdb->prefix . 'highest_payment_sessions', array( 'status' => 'paid_validated' ), array( 'paystack_ref' => $ref ) );
        return true;
    }

    private function send_order_to_dakazina( $order_id ) {
        global $wpdb;
        $order = $wpdb->get_row( $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}highest_orders WHERE id = %d", $order_id ) );
        if ( ! $order ) return false;
        // Network detection including AirtelTigo
        $network_id = 3;
        if ( stripos( $order->package, 'AirtelTigo' ) !== false ) {
            preg_match( '/(\d+)\s*GB/i', $order->package, $m );
            $gb         = (int) ( $m[1] ?? 0 );
            $network_id = ( $gb >= 15 ) ? 4 : 1;
        } elseif ( stripos( $order->package, 'MTN' ) !== false ) {
            $network_id = 3;
        } elseif ( stripos( $order->package, 'Telecel' ) !== false ) {
            $network_id = 2;
        } elseif ( stripos( $order->package, 'ATiShare' ) !== false ) {
            $network_id = 1;
        } elseif ( stripos( $order->package, 'ATBigTime' ) !== false ) {
            $network_id = 4;
        }
        preg_match( '/(\d+)GB/', $order->package, $matches );
        $shared_bundle = isset( $matches[1] ) ? (int) $matches[1] : 1;
        $payload = array(
            'recipient_msisdn' => $order->phone,
            'network_id'       => $network_id,
            'shared_bundle'    => $shared_bundle,
            'incoming_api_ref' => $order->ref,
        );
        $result = $this->dakazina_api_call( 'buy-data-package', 'POST', $payload );
        if ( ! isset( $result['error'] ) && isset( $result['data']['success'] ) && $result['data']['success'] ) {
            $dakazina_ref = $this->deep_find_ref( $result['data'] );
            $wpdb->update( $wpdb->prefix . 'highest_orders', array( 'incoming_api_ref' => $dakazina_ref ?: '', 'api_response' => wp_json_encode( $result ) ), array( 'id' => $order_id ) );
            return true;
        } else {
            $wpdb->update( $wpdb->prefix . 'highest_orders', array( 'status' => 'FAILED', 'api_response' => wp_json_encode( $result ) ), array( 'id' => $order_id ) );
            return false;
        }
    }


    /**
     * AJAX: Create an order after Paystack payment – verified via Paystack API,
     * no WordPress nonce required. Works for both walk-in and agent (Paystack path)
     * purchases. The session transient already holds all order metadata.
     */
    public function ajax_buy_data_verified() {
        $reference  = sanitize_text_field( $_POST['reference'] ?? '' );
        $trans_id   = sanitize_text_field( $_POST['trans']      ?? '' );
        $phone      = sanitize_text_field( $_POST['phone']      ?? '' );
        $email      = sanitize_email(      $_POST['email']      ?? '' );
        $nonce      = sanitize_text_field( $_POST['nonce']      ?? '' );

        // Cosmetic nonce check – never blocks
        if ( ! empty( $nonce ) && ! wp_verify_nonce( $nonce, 'highest_data_plug' ) ) {
            error_log( 'HighestDataPlug: ajax_buy_data_verified called with invalid nonce.' );
        }

        if ( empty( $reference ) || empty( $trans_id ) ) {
            wp_send_json_error( 'Missing payment data.' );
        }

        // Verify with Paystack
        $secret_key = get_option( 'highest_paystack_secret_key', '' );
        if ( empty( $secret_key ) ) {
            wp_send_json_error( 'Paystack not configured.' );
        }
        $verify = wp_remote_get( "https://api.paystack.co/transaction/verify/{$reference}", array(
            'headers' => array( 'Authorization' => 'Bearer ' . $secret_key ),
            'timeout' => 20,
        ));
        if ( is_wp_error( $verify ) ) {
            wp_send_json_error( 'Verification request failed.' );
        }
        $body = json_decode( wp_remote_retrieve_body( $verify ), true );
        if ( empty( $body['status'] ) || $body['data']['status'] !== 'success' ) {
            wp_send_json_error( 'Payment not successful.' );
        }
        if ( intval( $body['data']['id'] ) !== intval( $trans_id ) ) {
            wp_send_json_error( 'Transaction ID mismatch.' );
        }

        // Check if order already exists
        global $wpdb;
        $exists = $wpdb->get_var( $wpdb->prepare(
            "SELECT id FROM {$wpdb->prefix}highest_orders WHERE ref = %s", $reference
        ));
        if ( $exists ) {
            wp_send_json_success( 'Order already exists.' );
        }

        // Try to create the order via the normal session/transient path
        $order_created = $this->create_order_from_transient( $reference );

        if ( ! $order_created ) {
            // Fallback: use the order details sent directly by the browser
            $package_name  = sanitize_text_field( $_POST['package_name']  ?? '' );
            $network_id    = intval(              $_POST['network_id']     ?? 3 );
            $shared_bundle = intval(              $_POST['shared_bundle']  ?? 1 );
            $selling_price = floatval(            $_POST['selling_price']  ?? 0 );
            $agent_id      = intval(              $_POST['agent_id']       ?? 0 );
            $order_type    = sanitize_text_field( $_POST['order_type']     ?? 'walkin' );

            if ( empty( $package_name ) || $selling_price <= 0 ) {
                wp_send_json_error( 'Missing order details.' );
            }

            // Store a transient so the standard creation function can use it
            set_transient( 'hdp_order_' . $reference, array(
                'phone'         => $phone,
                'email'         => $email,
                'package_name'  => $package_name,
                'network_id'    => $network_id,
                'shared_bundle' => $shared_bundle,
                'selling_price' => $selling_price,
                'agent_id'      => $agent_id,
                'order_type'    => $order_type,
            ), 30 * MINUTE_IN_SECONDS );

            // Insert a payment session if none exists (for admin tracking)
            $existing_session = $wpdb->get_var( $wpdb->prepare(
                "SELECT id FROM {$wpdb->prefix}highest_payment_sessions WHERE paystack_ref = %s",
                $reference
            ));
            if ( ! $existing_session ) {
                $wpdb->insert( $wpdb->prefix . 'highest_payment_sessions', array(
                    'paystack_ref'   => $reference,
                    'amount'         => $selling_price,
                    'phone'          => $phone,
                    'email'          => $email,
                    'package_name'   => $package_name,
                    'network_id'     => $network_id,
                    'shared_bundle'  => $shared_bundle,
                    'selling_price'  => $selling_price,
                    'agent_id'       => $agent_id,
                    'order_type'     => $order_type,
                    'status'         => 'initiated',
                    'created_at'     => current_time( 'mysql' ),
                ) );
            }

            // Now call the standard creation again – this time it will succeed
            $order_created = $this->create_order_from_transient( $reference );
        }

        if ( $order_created ) {
            wp_send_json_success( 'Order placed successfully.' );
        } else {
            wp_send_json_error( 'Order creation failed.' );
        }
    }

    public function ajax_buy_data() {
        $this->verify_ajax_request();
        $ref = sanitize_text_field( $_POST['ref'] ?? '' );
        if ( $ref ) $this->create_order_from_transient( $ref );
        wp_send_json_success( 'Order placed' );
    }

    public function handle_paystack_webhook() {
        $input = file_get_contents( 'php://input' );
        $event = json_decode( $input, true );
        if ( ! $event || empty( $event['event'] ) || $event['event'] !== 'charge.success' ) {
            http_response_code( 200 );
            exit( 'Ignored' );
        }
        $data = $event['data'] ?? array();
        $reference = sanitize_text_field( $data['reference'] ?? '' );
        $secret_key = get_option( 'highest_paystack_secret_key', '' );
        if ( empty( $secret_key ) ) { http_response_code( 500 ); exit( 'No key' ); }
        $verify = wp_remote_get( "https://api.paystack.co/transaction/verify/{$reference}", array(
            'headers' => array( 'Authorization' => 'Bearer ' . $secret_key ),
            'timeout' => 20,
        ) );
        if ( is_wp_error( $verify ) ) { http_response_code( 500 ); exit( 'Verify error' ); }
        $body = json_decode( wp_remote_retrieve_body( $verify ), true );
        if ( empty( $body['status'] ) || $body['data']['status'] !== 'success' ) { http_response_code( 200 ); exit( 'Not success' ); }

        if ( $this->create_order_from_transient( $reference ) ) {
            http_response_code( 200 );
            exit( 'Order created' );
        }

        $email = sanitize_email( $data['customer']['email'] ?? '' );
        $user = get_user_by( 'email', $email );
        if ( ! $user || get_user_meta( $user->ID, 'is_highest_agent', true ) !== 'yes' ) {
            http_response_code( 200 );
            exit( 'Not agent' );
        }
        $amount = floatval( $data['amount'] ?? 0 ) / 100;
        $processed = get_user_meta( $user->ID, 'highest_processed_topups', true ) ?: array();
        if ( in_array( $reference, $processed ) ) {
            http_response_code( 200 );
            exit( 'Already processed' );
        }
        $wallet = (float) get_user_meta( $user->ID, 'highest_wallet', true );
        update_user_meta( $user->ID, 'highest_wallet', $wallet + $amount );
        $processed[] = $reference;
        update_user_meta( $user->ID, 'highest_processed_topups', $processed );

        global $wpdb;
        $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
            'user_id'   => $user->ID,
            'amount'    => $amount,
            'type'      => 'paystack_webhook',
            'status'    => 'approved',
            'reference' => $reference,
            'created_at'=> current_time( 'mysql' ),
        ) );
        $wpdb->update( $wpdb->prefix . 'highest_topup_history', array( 'balance_after' => $wallet + $amount ), array( 'id' => $wpdb->insert_id ) );

        if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user->ID ) ) {
            $phone = $this->get_agent_phone( $user->ID );
            if ( $phone ) $this->send_templated_sms( $phone, 'wallet_topup', array(
                'amount'  => number_format( $amount, 2 ),
                'balance' => number_format( $wallet + $amount, 2 ),
            ) );
        }
        http_response_code( 200 );
        exit( 'Wallet credited' );
    }

    public function register_paystack_webhook() {
        register_rest_route( 'highest/v1', '/paystack-webhook', array(
            'methods'             => 'POST',
            'callback'            => array( $this, 'handle_paystack_webhook' ),
            'permission_callback' => '__return_true',
        ) );
    }

    // ─── Moolre Webhook ───────────────────────────────────────────────────────

    public function register_moolre_webhook() {
        register_rest_route( 'highest/v1', '/moolre-webhook', array(
            'methods'             => 'POST',
            'callback'            => array( $this, 'handle_moolre_webhook' ),
            'permission_callback' => '__return_true',
        ) );
    }

    public function handle_moolre_webhook() {
        $input = file_get_contents( 'php://input' );
        $data  = json_decode( $input, true );

        // Log incoming webhook for debugging
        error_log( 'Moolre webhook received: ' . $input );

        $payload = $data['data'] ?? $data;
        $txstatus    = intval( $payload['txstatus'] ?? 0 );
        $externalref = sanitize_text_field( $payload['externalref'] ?? '' );

        if ( empty( $externalref ) ) {
            http_response_code( 400 );
            exit( 'No external reference' );
        }

        // Only process successful transactions (txstatus = 1)
        if ( $txstatus !== 1 ) {
            http_response_code( 200 );
            exit( 'Ignored – not successful' );
        }

        // Find the payment session
        global $wpdb;
        $session = $wpdb->get_row( $wpdb->prepare(
            "SELECT * FROM {$wpdb->prefix}highest_payment_sessions WHERE paystack_ref = %s",
            $externalref
        ) );

        if ( ! $session ) {
            http_response_code( 200 );
            exit( 'No session found' );
        }

        // Mark session as paid
        $wpdb->update(
            $wpdb->prefix . 'highest_payment_sessions',
            array( 'status' => 'paid_validated', 'updated_at' => current_time( 'mysql' ) ),
            array( 'id' => $session->id )
        );

        // Create the order
        $this->create_order_from_transient( $externalref );

        http_response_code( 200 );
        exit( 'OK' );
    }

    // ─── Dakazina Webhook (v5.1 – direct match + delivery SMS) ──
    public function handle_dakazina_webhook() {
    if ( ! isset( $_GET['hp_webhook'] ) || $_GET['hp_webhook'] !== 'callback' ) return;

    $input = file_get_contents( 'php://input' );
    $data  = json_decode( $input, true );

    // Log incoming webhook for debugging
    $log_dir = WP_CONTENT_DIR . '/highest-webhook-logs';
    if ( ! file_exists( $log_dir ) ) {
        wp_mkdir_p( $log_dir );
    }
    $log_file = $log_dir . '/dakazina-' . date( 'Y-m-d' ) . '.log';
    $log_entry = date( 'Y-m-d H:i:s' ) . ' | ' . $_SERVER['REMOTE_ADDR'] . ' | ' . $input . PHP_EOL;
    file_put_contents( $log_file, $log_entry, FILE_APPEND | LOCK_EX );

    update_option( 'highest_last_webhook_payload', array(
        'timestamp' => current_time( 'mysql' ),
        'raw'       => $input,
        'parsed'    => $data,
    ), false );

    if ( empty( $data ) ) { http_response_code( 200 ); exit; }

    // 1. Extract the raw status – keep it exactly as Dakazina sends
    $status = strtoupper( $data['status'] ?? $data['order_status'] ?? $data['delivery_status'] ?? 'PROCESSING' );

        // 2. Extract a Dakazina reference from the webhook data
    $incoming_ref = '';
    $keys = ['reference', 'incoming_api_ref', 'ref', 'transaction_id', 'id', 'order_code'];
    foreach ( $keys as $key ) {
        if ( ! empty( $data[ $key ] ) ) {
            $incoming_ref = sanitize_text_field( $data[ $key ] );
            break;
        }
    }

    // Normalise the customer phone from the webhook
    $raw_phone = $data['phone']
          ?? $data['msisdn']
          ?? $data['recipient']
          ?? $data['beneficiary_number']
          ?? $data['recipient_msisdn']
          ?? $data['metadata']['beneficiary_number']   // ← new
          ?? $data['metadata']['phone_number']         // ← new
          ?? '';
    $customer_phone = $this->normalise_phone( $raw_phone );

    global $wpdb;
    $table = $wpdb->prefix . 'highest_orders';

    // 3. Try to find the order – three fallback levels
    $order = null;

    // a) Exact match on incoming_api_ref (Dakazina transaction ID)
    if ( $incoming_ref ) {
        $order = $wpdb->get_row( $wpdb->prepare(
            "SELECT * FROM $table WHERE incoming_api_ref = %s LIMIT 1",
            $incoming_ref
        ) );
    }

    // b) Exact match on our own reference (stored in ref column)
    if ( ! $order && $incoming_ref ) {
        $order = $wpdb->get_row( $wpdb->prepare(
            "SELECT * FROM $table WHERE ref = %s LIMIT 1",
            $incoming_ref
        ) );
    }

    // c) Last resort: match by normalised phone only (latest order)
    if ( ! $order && ! empty( $customer_phone ) ) {
        $order = $wpdb->get_row( $wpdb->prepare(
            "SELECT * FROM $table WHERE phone = %s ORDER BY created_at DESC LIMIT 1",
            $customer_phone
        ) );
    }

    // Bail out cleanly if no matching order was found after all three attempts
    if ( ! $order ) {
        error_log( "Highest: Dakazina webhook could not find a matching order. Payload: " . print_r( $data, true ) );
        http_response_code( 200 );
        exit;
    }

    // Capture the current status before we overwrite it so the SMS guard works
    $old_status = strtoupper( $order->status ?? 'PROCESSING' );

    // 4. Update the order – always store the raw Dakazina status
    $wpdb->update(
        $table,
        array(
            'status'           => $status,
            'updated_at'       => current_time( 'mysql' ),
            'incoming_api_ref' => $incoming_ref ?: $order->incoming_api_ref,
        ),
        array( 'id' => $order->id ),
        array( '%s', '%s', '%s' ),
        array( '%d' )
    );

    // 5. Delivery SMS only if the new status contains a delivery‑like keyword
    $delivery_keywords = array( 'DELIVER', 'SUCCESS', 'COMPLETE', 'FULFILLED', 'SENT', 'DONE' );
    $is_delivered = false;
    foreach ( $delivery_keywords as $word ) {
        if ( stripos( $status, $word ) !== false ) {
            $is_delivered = true;
            break;
        }
    }

    if ( $this->is_sms_enabled() && $is_delivered && $old_status !== $status ) {
        if ( $order->user_id && ! $this->is_agent_sms_disabled( $order->user_id ) ) {
            $agent_phone = $this->get_agent_phone( $order->user_id );
            if ( $agent_phone ) {
                $this->send_templated_sms( $agent_phone, 'delivery_agent', array(
                    'package' => $order->package,
                    'phone'   => $order->phone,
                ) );
            }
        }
        if ( ! empty( $order->phone ) ) {
            $this->send_templated_sms( $order->phone, 'delivery_customer', array(
                'package' => $order->package,
            ) );
        }
    }

    http_response_code( 200 );
    exit;
}

    // ─── Merged Orders & Payments Admin Page ──
    public function orders_and_payments_page() {
        global $wpdb;
        $t_orders   = $wpdb->prefix . 'highest_orders';
        $t_sessions = $wpdb->prefix . 'highest_payment_sessions';

        // ── Handle bulk deletion of OLD initiated sessions (older than 1 hour) ──
        if ( isset( $_POST['cleanup_initiated'] ) && current_user_can( 'manage_options' ) ) {
            check_admin_referer( 'hdp_cleanup_initiated', '_wpnonce_cleanup_initiated' );
            $deleted = $wpdb->query( "DELETE FROM {$wpdb->prefix}highest_payment_sessions WHERE status = 'initiated' AND created_at < DATE_SUB(NOW(), INTERVAL 1 HOUR)" );
            echo '<div class="notice notice-success"><p>🧹 Deleted ' . intval( $deleted ) . ' old initiated session(s).</p></div>';
        }

        // ── Handle bulk cancellation of ALL pending/placed orders ────────────
        if ( isset( $_POST['cancel_all_pending'] ) && current_user_can( 'manage_options' ) ) {
            check_admin_referer( 'hdp_cancel_all_pending', '_wpnonce_cancel_pending' );
            $affected = $wpdb->query( $wpdb->prepare(
                "UPDATE {$wpdb->prefix}highest_orders
                 SET status = 'FAILED', api_response = %s, updated_at = NOW()
                 WHERE status IN ('PROCESSING', 'processing', 'PLACED', 'placed')",
                '{"note":"Admin bulk cancelled – not sent to DataKazina"}'
            ) );
            wp_redirect( admin_url( 'admin.php?page=highest-orders-payments&cancelled_pending=' . intval( $affected ) ) );
            exit;
        }

        // ── Handle UNCERTAIN order admin actions ─────────────────────────────
        if ( isset( $_POST['uncertain_order_id'] ) && current_user_can( 'manage_options' ) ) {
            check_admin_referer( 'hdp_uncertain_order_action', '_wpnonce_uncertain' );
            $order_id     = intval( $_POST['uncertain_order_id'] );
            $uncert_order = $wpdb->get_row( $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}highest_orders WHERE id = %d", $order_id ) );
            if ( $uncert_order && $uncert_order->status === 'UNCERTAIN' ) {
                if ( isset( $_POST['uncertain_resubmit'] ) ) {
                    $this->send_order_to_dakazina( $order_id );
                    echo '<div class="notice notice-success"><p>&#10003; Order resubmitted to DataKazina.</p></div>';
                } elseif ( isset( $_POST['uncertain_refund'] ) ) {
                    $wallet     = (float) get_user_meta( $uncert_order->user_id, 'highest_wallet', true );
                    $new_wallet = $wallet + (float) $uncert_order->agent_cost;
                    update_user_meta( $uncert_order->user_id, 'highest_wallet', $new_wallet );
                    $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                        'user_id'       => $uncert_order->user_id,
                        'amount'        => (float) $uncert_order->agent_cost,
                        'type'          => 'wallet_purchase_refund',
                        'status'        => 'approved',
                        'reference'     => $uncert_order->ref . '-admin-refund',
                        'balance_after' => $new_wallet,
                        'created_at'    => current_time( 'mysql' ),
                    ) );
                    $wpdb->update( $wpdb->prefix . 'highest_orders', array( 'status' => 'FAILED' ), array( 'id' => $order_id ) );
                    echo '<div class="notice notice-success"><p>&#10003; GH&#8373;' . number_format( $uncert_order->agent_cost, 2 ) . ' refunded to agent wallet. Order marked FAILED.</p></div>';
                } elseif ( isset( $_POST['uncertain_mark_delivered'] ) ) {
                    $this->update_order_status_by_ref( $uncert_order->ref, 'DELIVERED', array( 'api_response' => '{"note":"Admin manually marked DELIVERED from UNCERTAIN"}' ) );
                    $profit = round( (float) $uncert_order->selling_price - (float) $uncert_order->agent_cost, 2 );
                    if ( $profit > 0 && $uncert_order->user_id ) {
                        update_user_meta( $uncert_order->user_id, 'highest_profit', (float) get_user_meta( $uncert_order->user_id, 'highest_profit', true ) + $profit );
                        $this->add_profit_history( $uncert_order->user_id, $profit, 'profit_earned', $uncert_order->ref, "Admin resolved UNCERTAIN order – profit credited" );
                    }
                    echo '<div class="notice notice-success"><p>&#10003; Order marked as DELIVERED. Agent profit credited.</p></div>';
                }
            }
        }

        // -----------------------------------------------
        // Handle ORDER bulk actions (including delete)
        // MUST run before the DB queries below so that
        // deleted/updated orders are not fetched and then
        // re-displayed on the same page load.
        // -----------------------------------------------
        if ( isset( $_POST['bulk_apply'] ) && current_user_can( 'manage_options' ) ) {
            check_admin_referer( 'hdp_orders_bulk_action', '_wpnonce_orders_bulk' );
            $refs   = array_map( 'sanitize_text_field', $_POST['bulk_refs'] ?? array() );
            $action = sanitize_text_field( $_POST['bulk_action'] ?? '' );
            if ( ! empty( $refs ) && ! empty( $action ) ) {
                if ( $action === 'delete' ) {
                    $placeholders = implode( ',', array_fill( 0, count( $refs ), '%s' ) );
                    $wpdb->query( $wpdb->prepare(
                        "DELETE FROM {$wpdb->prefix}highest_orders WHERE ref IN ($placeholders)",
                        ...$refs
                    ) );
                    // Redirect so deleted orders don't reappear from a pre-deletion $orders fetch
                    wp_redirect( admin_url( 'admin.php?page=highest-orders-payments&bulk_done=' . count( $refs ) ) );
                    exit;
                } else {
                    foreach ( $refs as $ref ) {
                        switch ( $action ) {
                            case 'mark_delivered':
                                $this->update_order_status_by_ref( $ref, 'DELIVERED', array( 'api_response' => '{"note":"Admin bulk marked DELIVERED"}' ) );
                                break;
                            case 'retry':
                                $order_row = $wpdb->get_row( $wpdb->prepare( "SELECT id FROM {$wpdb->prefix}highest_orders WHERE ref = %s LIMIT 1", $ref ) );
                                if ( $order_row ) $this->send_order_to_dakazina( $order_row->id );
                                break;
                            case 'cancel':
                                $this->update_order_status_by_ref( $ref, 'FAILED', array( 'api_response' => '{"note":"Admin cancelled"}' ) );
                                break;
                        }
                    }
                    wp_redirect( admin_url( 'admin.php?page=highest-orders-payments&bulk_done=' . count( $refs ) ) );
                    exit;
                }
            }
        }

        // Search & filter params
        $search       = isset( $_GET['search'] ) ? sanitize_text_field( $_GET['search'] ) : '';
        $date_from    = isset( $_GET['date_from'] ) ? sanitize_text_field( $_GET['date_from'] ) : '';
        $date_to      = isset( $_GET['date_to'] ) ? sanitize_text_field( $_GET['date_to'] ) : '';
        $filter_agent = isset( $_GET['filter_agent'] ) ? intval( $_GET['filter_agent'] ) : 0;

        $where = "WHERE 1=1";
        if ( $search ) {
            $like   = '%' . $wpdb->esc_like( $search ) . '%';
            $where .= $wpdb->prepare( " AND (phone LIKE %s OR package LIKE %s OR ref LIKE %s)", $like, $like, $like );
        }
        if ( $date_from ) {
            $where .= $wpdb->prepare( " AND DATE(created_at) >= %s", $date_from );
        }
        if ( $date_to ) {
            $where .= $wpdb->prepare( " AND DATE(created_at) <= %s", $date_to );
        }
        if ( $filter_agent ) {
            $where .= $wpdb->prepare( " AND user_id = %d", $filter_agent );
        }

        // Orders – fetched AFTER bulk actions so the list always reflects current DB state
        $orders        = $wpdb->get_results( "SELECT * FROM $t_orders $where ORDER BY created_at DESC LIMIT 300" );
        $pending_count = (int) $wpdb->get_var( "SELECT COUNT(*) FROM $t_orders WHERE status IN ('PROCESSING','processing','PLACED','placed')" );
        $last_webhook  = get_option( 'highest_last_webhook_payload' );

        // Payment Sessions filter
        $filter   = isset( $_GET['status'] ) ? sanitize_text_field( $_GET['status'] ) : '';
        $where_s  = $filter ? $wpdb->prepare( "WHERE status = %s", $filter ) : '';
        $sessions = $wpdb->get_results( "SELECT * FROM $t_sessions $where_s ORDER BY created_at DESC LIMIT 300" );

        // -----------------------------------------------
        // 2. Handle SESSION bulk deletion
        // -----------------------------------------------
        if ( isset( $_POST['bulk_delete_sessions'] ) && current_user_can( 'manage_options' ) ) {
            check_admin_referer( 'hdp_bulk_delete_sessions', '_wpnonce_bulk_sessions' );
            $ids = array_map( 'intval', $_POST['session_ids'] ?? array() );
            if ( ! empty( $ids ) ) {
                $placeholders = implode( ',', array_fill( 0, count( $ids ), '%d' ) );
                $wpdb->query( $wpdb->prepare(
                    "DELETE FROM $t_sessions WHERE id IN ($placeholders)",
                    ...$ids
                ) );
                echo '<div class="notice notice-success"><p>✅ ' . count( $ids ) . ' session(s) deleted.</p></div>';
            }
        }
        // Single-session delete via GET (legacy fallback)
        if ( isset( $_GET['delete_session'] ) && current_user_can( 'manage_options' ) ) {
            $wpdb->delete( $t_sessions, array( 'id' => intval( $_GET['delete_session'] ) ) );
            echo '<div class="notice notice-success"><p>Session deleted.</p></div>';
        }
        ?>
        <div class="wrap">
            <h1>📡 HIGHEST DATA PLUG – Orders & Payments</h1>

            <!-- Tabs -->
            <div style="margin-bottom:20px;">
                <button class="hdp-tab-button active" onclick="switchTab('orders')">📦 Orders</button>
                <button class="hdp-tab-button" onclick="switchTab('payments')">💳 Payment Sessions</button>
            </div>

            <!-- Orders Tab -->
            <div id="tab-orders" class="hdp-tab-content">
                <?php if ( isset( $_GET['forced'] ) ) : ?>
                    <div class="notice notice-success"><p>✅ Order manually marked as DELIVERED.</p></div>
                <?php endif; ?>
                <?php if ( isset( $_GET['refund'] ) ) : ?>
                    <?php if ( $_GET['refund'] === 'success' ) : ?>
                        <div class="notice notice-success"><p>✅ Wallet refunded successfully. Order marked as REFUNDED.</p></div>
                    <?php else : ?>
                        <div class="notice notice-error"><p>❌ Refund failed – order not found, not FAILED, or already refunded.</p></div>
                    <?php endif; ?>
                <?php endif; ?>
                <?php if ( isset( $_GET['bulk_done'] ) ) : ?>
                    <div class="notice notice-success"><p>✅ Bulk action applied to <?php echo intval( $_GET['bulk_done'] ); ?> order(s).</p></div>
                <?php endif; ?>
                <?php if ( isset( $_GET['deleted'] ) ) : ?>
                    <div class="notice notice-success"><p>✅ <?php echo intval( $_GET['deleted'] ); ?> order(s) permanently deleted.</p></div>
                <?php endif; ?>
                <?php if ( isset( $_GET['queued'] ) ) : ?>
                    <div class="notice notice-success"><p>✅ Order queued for scheduled delivery.</p></div>
                <?php endif; ?>
                <?php if ( $last_webhook ) : ?>
                <div style="background:#fefce8;border:2px solid #ca8a04;padding:15px;margin:20px 0;border-radius:8px;">
                    <h2>🔔 Last Webhook Received from Dakazina</h2>
                    <p><strong>Time:</strong> <?php echo esc_html( $last_webhook['timestamp'] ?? '—' ); ?></p>
                    <pre style="background:#fff;padding:15px;border-radius:6px;max-height:300px;overflow:auto;font-size:12px;"><?php echo esc_html( print_r( $last_webhook['parsed'] ?? $last_webhook['raw'], true ) ); ?></pre>
                </div>
                <?php endif; ?>

                <!-- Search / Filter Form -->
                <form method="get" style="margin-bottom:15px; display:flex; flex-wrap:wrap; gap:8px; align-items:center;">
                    <input type="hidden" name="page" value="highest-orders-payments">
                    <input type="text" name="search" placeholder="Search phone / ref / package" value="<?php echo esc_attr( $search ); ?>" style="width:220px;">
                    <input type="date" name="date_from" value="<?php echo esc_attr( $date_from ); ?>">
                    <input type="date" name="date_to" value="<?php echo esc_attr( $date_to ); ?>">
                    <select name="filter_agent">
                        <option value="">All Agents</option>
                        <?php
                        $agent_list = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
                        foreach ( $agent_list as $ag ) {
                            echo '<option value="' . $ag->ID . '" ' . selected( $filter_agent, $ag->ID, false ) . '>' . esc_html( $ag->display_name ) . '</option>';
                        }
                        ?>
                    </select>
                    <button type="submit" class="button">Filter</button>
                    <?php if ( $search || $date_from || $date_to || $filter_agent ) : ?>
                        <a href="?page=highest-orders-payments" class="button">Clear</a>
                    <?php endif; ?>
                </form>

                <!-- Bulk Actions Wrapper -->
                <form method="post">
                <?php wp_nonce_field( 'hdp_orders_bulk_action', '_wpnonce_orders_bulk' ); ?>
                <table class="wp-list-table widefat fixed striped">
                    <thead><tr>
                        <th style="width:30px;"><input type="checkbox" id="check_all" onclick="document.querySelectorAll('.bulk-ref-check').forEach(c=>c.checked=this.checked)"></th>
                        <th>Date</th><th>Type</th><th>Agent</th><th>Phone</th><th>Package</th><th>Our Ref</th><th>API Ref</th><th>Status</th><th>API Response</th>
                    </tr></thead>
                    <tbody>
                    <?php foreach ( $orders as $order ) :
                        $agent     = $order->user_id ? get_user_by( 'id', $order->user_id ) : null;
                        $status    = strtoupper( $order->status ?? 'PROCESSING' );
                        $response  = $order->api_response ? json_decode( $order->api_response, true ) : array();
                        $resp_text = $response ? json_encode( $response, JSON_PRETTY_PRINT ) : 'No response yet';
                        ?>
                        <tr>
                            <td><input type="checkbox" class="bulk-ref-check" name="bulk_refs[]" value="<?php echo esc_attr( $order->ref ); ?>"></td>
                            <td><?php echo esc_html( $order->created_at ); ?></td>
                            <td><strong><?php echo esc_html( strtoupper( $order->order_type ) ); ?></strong></td>
                            <td><?php echo $agent ? esc_html( $agent->display_name ) : '—'; ?></td>
                            <td><?php echo esc_html( $order->phone ); ?></td>
                            <td><?php echo esc_html( $order->package ); ?></td>
                            <td><code style="font-size:10px;"><?php echo esc_html( $order->ref ); ?></code></td>
                            <td><code style="font-size:10px;color:<?php echo $order->incoming_api_ref ? '#2563eb' : '#9ca3af'; ?>;"><?php echo esc_html( $order->incoming_api_ref ?? '—' ); ?></code></td>
                            <td>
                                <?php
                                if ( $status === 'DELIVERED' ) {
                                    $badge_style = 'background:#10b981;color:white;';
                                } elseif ( $status === 'REFUNDED' ) {
                                    $badge_style = 'background:#6366f1;color:white;';
                                } elseif ( $status === 'UNCERTAIN' ) {
                                    $badge_style = 'background:#dc2626;color:white;';
                                } elseif ( $status === 'FAILED' ) {
                                    $badge_style = 'background:#ef4444;color:white;';
                                } else {
                                    $badge_style = 'background:#eab308;color:#1f2937;';
                                }
                                ?>
                                <span style="padding:5px 12px;border-radius:9999px;font-size:13px;<?php echo $badge_style; ?>"><?php echo esc_html( $status ); ?></span>

                                <?php if ( $status !== 'DELIVERED' && $status !== 'REFUNDED' && $status !== 'UNCERTAIN' ) : ?>
                                    <form method="post" style="display:inline-block;">
                                        <?php wp_nonce_field( 'hdp_force_delivered_' . $order->id, '_wpnonce_force_delivered' ); ?>
                                        <input type="hidden" name="force_delivered_ref" value="<?php echo esc_attr( $order->ref ); ?>">
                                        <input type="hidden" name="force_delivered_order_id" value="<?php echo (int) $order->id; ?>">
                                        <button type="submit" name="force_delivered" class="button button-small button-primary" onclick="return confirm('Force this order to show as DELIVERED?');">Mark Delivered</button>
                                    </form>
                                <?php endif; ?>

                                <?php if ( $status === 'FAILED' && $order->order_type === 'agent' && $order->agent_cost > 0 ) : ?>
                                    <form method="post" style="display:inline-block;margin-left:4px;">
                                        <?php wp_nonce_field( 'hdp_refund_wallet_' . $order->id, '_wpnonce_refund_wallet' ); ?>
                                        <input type="hidden" name="refund_wallet_ref" value="<?php echo esc_attr( $order->ref ); ?>">
                                        <input type="hidden" name="refund_wallet_order_id" value="<?php echo (int) $order->id; ?>">
                                        <button type="submit" name="refund_wallet" class="button button-small" style="background:#f59e0b;color:#fff;border-color:#f59e0b;" onclick="return confirm('Refund GH&#8373;<?php echo number_format( $order->agent_cost, 2 ); ?> back to the agent wallet?');">Refund Wallet</button>
                                    </form>
                                <?php endif; ?>

                                <?php if ( $status === 'UNCERTAIN' && $order->user_id ) : ?>
                                    <form method="post" style="display:inline-block;margin-left:4px;">
                                        <?php wp_nonce_field( 'hdp_uncertain_order_action', '_wpnonce_uncertain' ); ?>
                                        <input type="hidden" name="uncertain_order_id" value="<?php echo (int) $order->id; ?>">
                                        <button type="submit" name="uncertain_resubmit" class="button button-small" style="background:#2563eb;color:#fff;border-color:#2563eb;" title="Resubmit to DataKazina">Resubmit</button>
                                        <button type="submit" name="uncertain_refund" class="button button-small" style="background:#f59e0b;color:#fff;border-color:#f59e0b;" onclick="return confirm('Refund GH&#8373;<?php echo number_format( $order->agent_cost, 2 ); ?> to agent wallet?');" title="Refund agent wallet">Refund</button>
                                        <button type="submit" name="uncertain_mark_delivered" class="button button-small" style="background:#10b981;color:#fff;border-color:#10b981;" onclick="return confirm('Mark this order as DELIVERED and credit agent profit?');" title="Mark as delivered manually">Delivered</button>
                                    </form>
                                <?php endif; ?>
                            </td>
                            <td><pre style="font-size:11px;max-height:120px;overflow:auto;"><?php echo esc_html( $resp_text ); ?></pre></td>
                        </tr>
                    <?php endforeach; ?>
                    </tbody>
                </table>
                <div style="margin-top:12px; display:flex; gap:8px; align-items:center;">
                    <select name="bulk_action">
                        <option value="">Bulk Action</option>
                        <option value="mark_delivered">Mark Delivered</option>
                        <option value="retry">Retry Delivery</option>
                        <option value="cancel">Cancel</option>
                        <option value="delete">Delete</option>
                    </select>
                    <button type="submit" name="bulk_apply" class="button">Apply</button>
                </div>
                </form>

                <?php if ( isset( $_GET['cancelled_pending'] ) ) : ?>
                    <div class="notice notice-success"><p>❌ <?php echo intval( $_GET['cancelled_pending'] ); ?> pending/placed order(s) marked as FAILED and will not be sent to DataKazina.</p></div>
                <?php endif; ?>

                <?php if ( $pending_count > 0 ) : ?>
                <form method="post" style="margin-top:10px;">
                    <?php wp_nonce_field( 'hdp_cancel_all_pending', '_wpnonce_cancel_pending' ); ?>
                    <button type="submit" name="cancel_all_pending" class="button button-secondary"
                            style="border-color:#dc2626;color:#dc2626;"
                            onclick="return confirm('Mark ALL <?php echo $pending_count; ?> pending/placed orders as FAILED?\n\nThis prevents them from being sent to DataKazina. You can delete them later using bulk delete.');">
                        ❌ Cancel All Pending Orders (<?php echo $pending_count; ?>)
                    </button>
                </form>
                <?php endif; ?>
            </div>

            <!-- Payment Sessions Tab -->
            <div id="tab-payments" class="hdp-tab-content" style="display:none;">
                <form method="get" style="margin-bottom:15px;">
                    <input type="hidden" name="page" value="highest-orders-payments">
                    <select name="status">
                        <option value="">All</option>
                        <option value="initiated" <?php selected( $filter, 'initiated' ); ?>>Initiated</option>
                        <option value="paid_validated" <?php selected( $filter, 'paid_validated' ); ?>>Paid & Validated</option>
                        <option value="failed" <?php selected( $filter, 'failed' ); ?>>Failed</option>
                    </select>
                    <button type="submit" class="button">Filter</button>
                </form>
                <form method="post" style="margin-bottom:15px;">
                    <?php wp_nonce_field( 'hdp_cleanup_initiated', '_wpnonce_cleanup_initiated' ); ?>
                    <button type="submit" name="cleanup_initiated" class="button" onclick="return confirm('Delete ALL initiated sessions older than 1 hour? This cannot be undone.');">
                        🧹 Delete Old Initiated Sessions (1h+)
                    </button>
                </form>
                <form method="post" id="bulk-sessions-form">
                    <?php wp_nonce_field( 'hdp_bulk_delete_sessions', '_wpnonce_bulk_sessions' ); ?>
                    <table class="wp-list-table widefat fixed striped">
                        <thead><tr>
                            <th style="width:30px;"><input type="checkbox" id="check_all_sessions" onclick="document.querySelectorAll('.session-cb').forEach(cb => cb.checked = this.checked);"></th>
                            <th>ID</th><th>Ref</th><th>Amount</th><th>Phone</th><th>Package</th><th>Agent</th><th>Status</th><th>Date</th><th>Action</th>
                        </tr></thead>
                        <tbody>
                        <?php foreach ( $sessions as $s ) :
                            $agent = $s->agent_id ? get_user_by( 'id', $s->agent_id ) : null;
                        ?>
                            <tr>
                                <td><input type="checkbox" class="session-cb" name="session_ids[]" value="<?php echo (int) $s->id; ?>"></td>
                                <td><?php echo $s->id; ?></td>
                                <td><code><?php echo esc_html( $s->paystack_ref ); ?></code></td>
                                <td>GH₵<?php echo number_format( $s->amount, 2 ); ?></td>
                                <td><?php echo esc_html( $s->phone ); ?></td>
                                <td><?php echo esc_html( $s->package_name ?: '—' ); ?></td>
                                <td><?php echo $agent ? esc_html( $agent->display_name ) : '—'; ?></td>
                                <td><span style="padding:4px 12px;border-radius:9999px;font-size:12px;background:<?php echo $s->status === 'paid_validated' ? '#10b981' : ( $s->status === 'failed' ? '#ef4444' : '#eab308' ); ?>;color:white;"><?php echo esc_html( $s->status ); ?></span></td>
                                <td><?php echo $s->created_at; ?></td>
                                <td style="white-space:nowrap;">
                                    <?php if ( $s->status === 'initiated' ) : ?>
                                        <button class="button button-small verify-session" data-id="<?php echo $s->id; ?>">Verify & Create Order</button>
                                        <button class="button button-small verify-payment-admin" data-id="<?php echo $s->id; ?>">Verify Payment</button>
                                        <button class="button button-small retry-sms-btn" data-id="<?php echo $s->id; ?>" style="background:#6366f1;color:#fff;border-color:#6366f1;">Send Retry SMS</button>
                                        <?php if ( $s->agent_id ) : ?>
                                        <button class="button button-small notify-agent-btn" data-id="<?php echo $s->id; ?>" style="background:#f59e0b;color:#fff;border-color:#f59e0b;">Notify Agent</button>
                                        <?php endif; ?>
                                    <?php endif; ?>
                                    <a href="?page=highest-orders-payments&delete_session=<?php echo $s->id; ?>" class="button button-small" onclick="return confirm('Delete this session?')">Delete</a>
                                </td>
                            </tr>
                        <?php endforeach; ?>
                        </tbody>
                    </table>
                    <p style="margin-top:12px;">
                        <button type="submit" name="bulk_delete_sessions" class="button button-primary" onclick="return confirm('Delete all selected sessions? This cannot be undone.');">Delete Selected</button>
                    </p>
                </form>
            </div>
        </div>
        <style>
            .hdp-tab-button { padding: 10px 16px; border: none; background: #f1f5f9; cursor: pointer; font-size: 14px; border-radius: 8px 8px 0 0; margin-right: 4px; }
            .hdp-tab-button.active { background: #fff; font-weight: bold; border: 1px solid #ccd0d4; border-bottom: none; }
            .hdp-tab-content { background: #fff; padding: 20px; border-radius: 0 8px 8px 8px; box-shadow: 0 1px 4px rgba(0,0,0,0.05); }
        </style>
        <script>
        function switchTab(tab) {
            document.querySelectorAll('.hdp-tab-content').forEach(el => el.style.display = 'none');
            document.querySelectorAll('.hdp-tab-button').forEach(el => el.classList.remove('active'));
            document.getElementById('tab-' + tab).style.display = 'block';
            event.target.classList.add('active');
        }
        // Payment session actions
        jQuery(document).ready(function($) {
            $('.verify-session').on('click', function() {
                var btn = $(this), id = btn.data('id');
                btn.text('Verifying...').prop('disabled', true);

                // Step 1: verify via v2
                var fd = new FormData();
                fd.append('action', 'highest_verify_payment_session_v2');
                fd.append('nonce', '<?php echo wp_create_nonce("highest_data_plug"); ?>');
                fd.append('session_id', id);

                fetch(ajaxurl, { method:'POST', credentials:'same-origin', body:fd })
                .then(r => r.json())
                .then(res => {
                    if (res.success) {
                        if (res.data.status === 'order_exists') {
                            alert(res.data.message);
                            location.reload();
                            return;
                        }
                        // Verified – ask to create order
                        if (confirm('✅ Payment verified. Create data order for this session?')) {
                            var fd2 = new FormData();
                            fd2.append('action', 'highest_create_order_for_session');
                            fd2.append('nonce', '<?php echo wp_create_nonce("highest_data_plug"); ?>');
                            fd2.append('session_id', id);
                            fetch(ajaxurl, { method:'POST', credentials:'same-origin', body:fd2 })
                            .then(r => r.json())
                            .then(res2 => {
                                if (res2.success) alert('✅ Order created!');
                                else alert('❌ ' + (res2.data || 'Order creation failed'));
                                location.reload();
                            });
                        } else {
                            btn.text('Verify & Create Order').prop('disabled', false);
                        }
                    } else {
                        alert('❌ ' + (res.data || 'Verification failed'));
                        btn.text('Verify & Create Order').prop('disabled', false);
                    }
                })
                .catch(() => {
                    alert('Network error');
                    btn.text('Verify & Create Order').prop('disabled', false);
                });
            });

            $('.verify-payment-admin').on('click', function() {
                if (!confirm('Mark as verified without creating order?')) return;
                var btn = $(this), id = btn.data('id');
                btn.text('Checking...').prop('disabled', true);
                var fd = new FormData();
                fd.append('action', 'highest_admin_verify_payment');
                fd.append('nonce', '<?php echo wp_create_nonce( "highest_data_plug" ); ?>');
                fd.append('session_id', id);
                fetch(ajaxurl, { method:'POST', credentials:'same-origin', body:fd })
                .then(r => r.json()).then(res => {
                    if (res.success) { alert(res.data); location.reload(); }
                    else { alert('Error: ' + (res.data || 'Unknown')); btn.text('Verify Payment').prop('disabled', false); }
                });
            });

            // Send Retry SMS button
            $('.retry-sms-btn').on('click', function() {
                var btn = $(this), id = btn.data('id');
                btn.prop('disabled', true).text('Sending...');
                var fd = new FormData();
                fd.append('action', 'highest_send_retry_sms');
                fd.append('nonce', '<?php echo wp_create_nonce( "highest_data_plug" ); ?>');
                fd.append('session_id', id);
                fetch(ajaxurl, { method:'POST', credentials:'same-origin', body:fd })
                .then(r => r.json())
                .then(res => {
                    if (res.success) { alert('✅ ' + res.data); }
                    else { alert('❌ ' + (res.data || 'Failed')); }
                })
                .finally(() => { btn.prop('disabled', false).text('Send Retry SMS'); });
            });

            // Notify Agent button
            $('.notify-agent-btn').on('click', function() {
                var btn = $(this), id = btn.data('id');
                btn.prop('disabled', true).text('Notifying...');
                var fd = new FormData();
                fd.append('action', 'highest_notify_agent');
                fd.append('nonce', '<?php echo wp_create_nonce( "highest_data_plug" ); ?>');
                fd.append('session_id', id);
                fetch(ajaxurl, { method:'POST', credentials:'same-origin', body:fd })
                .then(r => r.json())
                .then(res => {
                    if (res.success) { alert('✅ ' + res.data); }
                    else { alert('❌ ' + (res.data || 'Failed')); }
                })
                .finally(() => { btn.prop('disabled', false).text('Notify Agent'); });
            });
        });
        </script>
        <?php
    }

    // ─── Cron jobs ─────────────────────────────────────────
    public function cron_check_payment_sessions() {
        global $wpdb;
        $table = $wpdb->prefix . 'highest_payment_sessions';
        $sessions = $wpdb->get_results( "SELECT * FROM $table WHERE status='initiated' AND created_at < DATE_SUB(NOW(), INTERVAL 5 MINUTE) LIMIT 20" );
        foreach ( $sessions as $s ) {
            $this->create_order_from_transient( $s->paystack_ref );
        }
        $transients = $wpdb->get_col( "SELECT option_name FROM {$wpdb->options} WHERE option_name LIKE '%_transient_hdp_order_%'" );
        foreach ( $transients as $opt ) {
            $ref = str_replace( '_transient_hdp_order_', '', $opt );
            $this->create_order_from_transient( $ref );
        }
    }

    public function cron_auto_check_orders() {
    global $wpdb;
    $table = $wpdb->prefix . 'highest_orders';
    $orders = $wpdb->get_results( "SELECT ref, incoming_api_ref FROM $table WHERE status IN ('PROCESSING', 'processing', 'PLACED', 'placed')" );
    foreach ( $orders as $order ) {
        $check_ref = ! empty( $order->incoming_api_ref ) ? $order->incoming_api_ref : $order->ref;
        $result = $this->dakazina_api_call( 'fetch-single-transaction', 'POST', array( 'transaction_id' => $check_ref ) );
        if ( ! isset( $result['error'] ) ) {
            $raw_status = $this->deep_find_status( $result['data'] ?? array() );
            if ( $raw_status !== '' ) {
                // Keep raw status – never normalise
                $new_status = strtoupper( trim( $raw_status ) );
                $this->update_order_status_by_ref( $check_ref, $new_status, array( 'api_response' => wp_json_encode( $result ) ) );
            }
        }
        usleep( 100000 );
    }

    // Low-balance alert
    if ( get_option( 'highest_low_balance_alert_enabled', 'no' ) === 'yes' ) {
        $balance_res = $this->dakazina_api_call( 'check-console-balance', 'GET' );
        if ( ! isset( $balance_res['error'] ) && isset( $balance_res['data'] ) ) {
            $current_balance = floatval( $balance_res['data']['balance'] ?? $balance_res['data']['wallet'] ?? 0 );
            $threshold = floatval( get_option( 'highest_dk_balance_threshold', 0 ) );
            if ( $threshold > 0 && $current_balance <= $threshold ) {
                $phone = get_option( 'highest_emergency_sms_phone', '' );
                if ( $phone ) {
                    $this->send_sms( $phone, "ALERT: DataKazina balance is down to GH₵{$current_balance}. Please top up.", 'emergency_admin' );
                }
            }
        }
    }
}

    private function normalise_delivery_status( $raw ) {
        $s = strtoupper( trim( (string) $raw ) );
        if ( stripos( $s, 'DELIVER' ) !== false || stripos( $s, 'SUCCESS' ) !== false || stripos( $s, 'COMPLETE' ) !== false || stripos( $s, 'RECEIVED' ) !== false || stripos( $s, 'SENT' ) !== false || stripos( $s, 'DONE' ) !== false || stripos( $s, 'APPROVE' ) !== false || stripos( $s, 'FULFILLED' ) !== false ) return 'DELIVERED';
        if ( stripos( $s, 'FAIL' ) !== false || stripos( $s, 'ERROR' ) !== false || stripos( $s, 'CANCEL' ) !== false || stripos( $s, 'REJECT' ) !== false ) return 'FAILED';
        return $s ?: 'PROCESSING';
    }

    private function deep_find_status( $data ) {
        if ( ! is_array( $data ) ) return '';
        $keys = array( 'status', 'order_status', 'delivery_status', 'deliverystatus', 'orderStatus', 'deliveryStatus', 'state' );
        foreach ( $keys as $k ) { if ( isset( $data[ $k ] ) && (string) $data[ $k ] !== '' ) return (string) $data[ $k ]; }
        foreach ( $data as $v ) { if ( is_array( $v ) ) { $found = $this->deep_find_status( $v ); if ( $found !== '' ) return $found; } }
        return '';
    }

    private function deep_find_ref( $data ) {
        if ( ! is_array( $data ) ) return '';
        $keys = array( 'reference', 'ref', 'transaction_id', 'transactionId', 'trxn_id', 'order_id', 'orderId', 'order_code', 'id', 'incoming_api_ref' );
        foreach ( $keys as $k ) { if ( isset( $data[ $k ] ) && (string) $data[ $k ] !== '' ) return (string) $data[ $k ]; }
        foreach ( $data as $v ) { if ( is_array( $v ) ) { $found = $this->deep_find_ref( $v ); if ( $found !== '' ) return $found; } }
        return '';
    }

    public function handle_daily_login_bonus( $user_login, $user ) {
        if ( get_user_meta( $user->ID, 'is_highest_agent', true ) !== 'yes' ) return;
        $today = date( 'Y-m-d' );
        $last = get_user_meta( $user->ID, 'highest_last_daily_bonus', true );
        if ( $last === $today ) return;
        $bonus = floatval( get_option( 'highest_daily_bonus_amount', 0.5 ) );
        if ( $bonus > 0 ) {
            $wallet = (float) get_user_meta( $user->ID, 'highest_wallet', true );
            update_user_meta( $user->ID, 'highest_wallet', $wallet + $bonus );
            global $wpdb;
            $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                'user_id' => $user->ID, 'amount' => $bonus, 'type' => 'daily_login_bonus', 'status' => 'approved', 'reference' => 'DAILY-' . date( 'Ymd' ) . '-' . $user->ID, 'created_at'=> current_time('mysql'),
            ) );
            $wpdb->update( $wpdb->prefix . 'highest_topup_history', array( 'balance_after' => $wallet + $bonus ), array( 'id' => $wpdb->insert_id ) );
        }
        update_user_meta( $user->ID, 'highest_last_daily_bonus', $today );
        $streak = (int) get_user_meta( $user->ID, 'highest_login_streak', true );
        $yesterday = date( 'Y-m-d', strtotime( '-1 day' ) );
        if ( $last === $yesterday ) $streak++; else $streak = 1;
        update_user_meta( $user->ID, 'highest_login_streak', $streak );
    }

    // ─── Admin Menu (Merged Orders & Payments replaces Delivery Portal + Payment Sessions) ──
    public function add_admin_menu() {
        add_menu_page(
            'HIGHEST DATA PLUG',
            'HIGHEST DATA PLUG',
            'manage_options',
            'highest-data-plug',
            array( $this, 'admin_dashboard_page' ),
            'dashicons-mobile',
            58
        );
        add_submenu_page( 'highest-data-plug', 'Dashboard', 'Dashboard', 'manage_options', 'highest-data-plug', array( $this, 'admin_dashboard_page' ) );
        add_submenu_page( 'highest-data-plug', 'Settings', 'Settings', 'manage_options', 'highest-settings', array( $this, 'settings_page' ) );
        add_submenu_page( 'highest-data-plug', 'Orders & Payments', 'Orders & Payments', 'manage_options', 'highest-orders-payments', array( $this, 'orders_and_payments_page' ) );
        add_submenu_page( 'highest-data-plug', 'Withdrawals', 'Withdrawals', 'manage_options', 'highest-withdrawals', array( $this, 'withdrawals_page' ) );
        add_submenu_page( 'highest-data-plug', 'All Agents', 'All Agents', 'manage_options', 'highest-all-agents', array( $this, 'all_agents_page' ) );
        add_submenu_page( 'highest-data-plug', 'Manual Top-ups', 'Manual Top-ups', 'manage_options', 'highest-manual-topups', array( $this, 'manual_topups_page' ) );
        add_submenu_page( 'highest-data-plug', 'Top-up History', 'Top-up History', 'manage_options', 'highest-topup-history', array( $this, 'topup_history_page' ) );
        add_submenu_page( 'highest-data-plug', 'Announcements', 'Announcements', 'manage_options', 'highest-announcements', array( $this, 'announcements_page' ) );
        add_submenu_page( 'highest-data-plug', 'Bulk SMS (Agents)', 'Bulk SMS (Agents)', 'manage_options', 'highest-bulk-sms', array( $this, 'bulk_sms_page' ) );
        add_submenu_page( 'highest-data-plug', 'Bulk SMS to Customers', 'Bulk SMS to Customers', 'manage_options', 'highest-bulk-sms-customers', array( $this, 'bulk_sms_customers_page' ) );
        add_submenu_page( 'highest-data-plug', 'Profit History', 'Profit History', 'manage_options', 'highest-profit-history', array( $this, 'profit_history_page' ) );
        add_submenu_page( 'highest-data-plug', 'AFA Registrations', 'AFA Registrations', 'manage_options', 'highest-afa', array( $this, 'afa_admin_page' ) );
        add_submenu_page( 'highest-data-plug', 'Agent Referrals', 'Agent Referrals', 'manage_options', 'highest-referrals', array( $this, 'referral_admin_page' ) );
        add_submenu_page( 'highest-data-plug', 'Package Stock', 'Package Stock', 'manage_options', 'highest-package-stock', array( $this, 'package_stock_page' ) );
        add_submenu_page( 'highest-data-plug', 'Scheduled SMS', 'Scheduled SMS', 'manage_options', 'highest-scheduled-sms', array( $this, 'scheduled_sms_page' ) );
        add_submenu_page( 'highest-data-plug', 'System Health', 'System Health', 'manage_options', 'highest-system-health', array( $this, 'system_health_page' ) );
        add_submenu_page( 'highest-data-plug', 'WAEC Vouchers', 'WAEC Vouchers', 'manage_options', 'highest-waec-vouchers', array( $this, 'waec_vouchers_page' ) );
        add_submenu_page( 'highest-data-plug', 'WAEC Orders', 'WAEC Orders', 'manage_options', 'highest-waec-orders', array( $this, 'waec_orders_page' ) );
    }

    // ─── Admin Master Dashboard (unchanged from v4.2.1) ──
    public function admin_dashboard_page() {
        global $wpdb;
        $table_orders = $wpdb->prefix . 'highest_orders';
        $table_topup  = $wpdb->prefix . 'highest_topup_history';

        $today_sales = $wpdb->get_var( "SELECT SUM(selling_price) FROM $table_orders WHERE DATE(created_at) = CURDATE() AND status = 'DELIVERED'" ) ?: 0;
        $total_agents = count( get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes', 'fields' => 'ID' ) ) );
        $pending_withdrawals = 0;
        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        foreach ( $agents as $agent ) {
            $reqs = get_user_meta( $agent->ID, 'highest_withdraw_requests', true ) ?: array();
            foreach ( $reqs as $r ) {
                if ( isset( $r['status'] ) && $r['status'] === 'pending_approval' ) $pending_withdrawals++;
            }
        }
        $recent_orders = $wpdb->get_results( "SELECT * FROM $table_orders ORDER BY created_at DESC LIMIT 10" );
        $top_agents = $wpdb->get_results( "SELECT user_id, SUM(selling_price) as total FROM $table_orders WHERE status='DELIVERED' GROUP BY user_id ORDER BY total DESC LIMIT 5" );
        $today_orders_count = $wpdb->get_var( $wpdb->prepare( "SELECT COUNT(*) FROM $table_orders WHERE DATE(created_at) = %s", date('Y-m-d') ) ) ?: 0;
        $pending_count = $wpdb->get_var( "SELECT COUNT(*) FROM $table_orders WHERE status = 'PROCESSING' OR status = 'processing'" ) ?: 0;

        ?>
        <div class="wrap">
            <h1 style="font-size:28px; margin-bottom:20px;">📊 HIGHEST DATA PLUG – Master Dashboard</h1>
            <div style="display:grid; grid-template-columns: repeat(auto-fit, minmax(180px,1fr)); gap:20px; margin-bottom:30px;">
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <p style="font-size:14px; color:#64748b; margin-bottom:8px;">Today's Sales</p>
                    <p style="font-size:32px; font-weight:bold; color:#10b981;">GH₵<?php echo number_format( $today_sales, 2 ); ?></p>
                </div>
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <p style="font-size:14px; color:#64748b; margin-bottom:8px;">Active Agents</p>
                    <p style="font-size:32px; font-weight:bold; color:#3b82f6;"><?php echo $total_agents; ?></p>
                </div>
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <p style="font-size:14px; color:#64748b; margin-bottom:8px;">Pending Withdrawals</p>
                    <p style="font-size:32px; font-weight:bold; color:#f59e0b;"><?php echo $pending_withdrawals; ?></p>
                </div>
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <p style="font-size:14px; color:#64748b; margin-bottom:8px;">Today's Orders</p>
                    <p style="font-size:32px; font-weight:bold; color:#3b82f6;"><?php echo $today_orders_count; ?></p>
                </div>
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <p style="font-size:14px; color:#64748b; margin-bottom:8px;">Pending Orders</p>
                    <p style="font-size:32px; font-weight:bold; color:#f59e0b;"><?php echo $pending_count; ?></p>
                </div>
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <p style="font-size:14px; color:#64748b; margin-bottom:8px;">Quick Actions</p>
                    <div style="display:flex; flex-wrap:wrap; gap:8px; margin-top:12px;">
                        <a href="<?php echo admin_url('admin.php?page=highest-withdrawals'); ?>" class="button" style="font-size:12px;">Withdrawals</a>
                        <a href="<?php echo admin_url('admin.php?page=highest-all-agents'); ?>" class="button" style="font-size:12px;">Manage Agents</a>
                        <a href="<?php echo admin_url('admin.php?page=highest-settings'); ?>" class="button" style="font-size:12px;">Settings</a>
                        <a href="<?php echo admin_url('admin.php?page=highest-system-health'); ?>" class="button" style="font-size:12px;">System Health</a>
                    </div>
                </div>
            </div>

            <div style="display:grid; grid-template-columns: 2fr 1fr; gap:20px;">
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <h3 style="margin-bottom:15px; font-size:18px;">📦 Recent Orders</h3>
                    <table class="wp-list-table widefat fixed striped" style="border:0;">
                        <thead><tr><th>Date</th><th>Phone</th><th>Package</th><th>Status</th></tr></thead>
                        <tbody>
                        <?php foreach ( $recent_orders as $o ): 
                            $st = strtoupper($o->status ?? 'PROCESSING');
                            $color = $st === 'DELIVERED' ? '#10b981' : ($st === 'FAILED' ? '#ef4444' : '#eab308');
                        ?>
                            <tr>
                                <td><?php echo date('d/m H:i', strtotime($o->created_at)); ?></td>
                                <td><?php echo esc_html($o->phone); ?></td>
                                <td><?php echo esc_html($o->package); ?></td>
                                <td><span style="background:<?php echo $color; ?>; color:white; padding:2px 10px; border-radius:999px; font-size:12px;"><?php echo $st; ?></span></td>
                            </tr>
                        <?php endforeach; ?>
                        </tbody>
                    </table>
                </div>
                <div style="background:#fff; padding:20px; border-radius:16px; box-shadow:0 4px 12px rgba(0,0,0,0.05);">
                    <h3 style="margin-bottom:15px; font-size:18px;">🏆 Top Agents</h3>
                    <ul style="list-style:none; margin:0; padding:0;">
                        <?php foreach ( $top_agents as $ta ): 
                            $agent = get_user_by('id', $ta->user_id);
                        ?>
                            <li style="padding:10px 0; border-bottom:1px solid #f1f5f9; display:flex; justify-content:space-between;">
                                <span><?php echo $agent ? esc_html($agent->display_name) : 'Agent #'.$ta->user_id; ?></span>
                                <strong>GH₵<?php echo number_format($ta->total, 2); ?></strong>
                            </li>
                        <?php endforeach; ?>
                    </ul>
                </div>
            </div>
        </div>
        <?php
    }

    // ─── Settings Page (updated with split EMSG toggles) ──
    public function settings_page() {
        if ( isset( $_POST['highest_save'] ) ) {
            update_option( 'highest_ussd_agent_prefix', sanitize_text_field( $_POST['ussd_agent_prefix'] ?? '110' ) );
            update_option( 'highest_ussd_shortcode', sanitize_text_field( $_POST['ussd_shortcode'] ?? '*919*909*' ) );
            update_option( 'highest_ussd_payment_gateway', sanitize_text_field( $_POST['ussd_payment_gateway'] ?? '' ) );
            update_option( 'highest_paystack_public_key', sanitize_text_field( $_POST['paystack_public'] ?? '' ) );
            update_option( 'highest_paystack_secret_key', sanitize_text_field( $_POST['paystack_secret'] ?? '' ) );
            update_option( 'highest_dakazina_api_key', sanitize_text_field( $_POST['dakazina_api_key'] ?? '' ) );
            update_option( 'highest_disable_public_buy', ! empty( $_POST['disable_public_buy'] ) );
            update_option( 'highest_disable_agent_buy', ! empty( $_POST['disable_agent_buy'] ) );
            update_option( 'highest_disable_all_buy', ! empty( $_POST['disable_all_buy'] ) );
            update_option( 'highest_disable_marketplace_banner', ! empty( $_POST['disable_marketplace_banner'] ) );
            update_option( 'highest_show_whatsapp_banner', isset( $_POST['show_whatsapp_banner'] ) ? 'yes' : 'no' );
            update_option( 'highest_whatsapp_number', sanitize_text_field( $_POST['whatsapp_number'] ?? '0551907463' ) );
            update_option( 'highest_community_link', esc_url_raw( $_POST['community_link'] ?? '' ) );
            update_option( 'highest_daily_bonus_amount', floatval( $_POST['daily_bonus_amount'] ?? 0.5 ) );
            update_option( 'highest_sms_provider', sanitize_text_field( $_POST['sms_provider'] ?? 'hubtel' ) );
            // Per-message-type enable flags and provider selection
            $sms_message_types = array(
                'order_received', 'delivery_customer', 'delivery_agent', 'order_placed_agent', 'wallet_topup',
                'withdrawal_update', 'manual_topup_agent', 'welcome_agent', 'emergency_admin',
                'retry_sms', 'notify_agent', 'scheduled_campaign', 'bulk_agents', 'bulk_customers',
            );
            foreach ( $sms_message_types as $type ) {
                update_option( "highest_sms_enabled_{$type}", isset( $_POST["sms_enabled_{$type}"] ) ? 'yes' : 'no' );
                update_option( "highest_sms_provider_{$type}", sanitize_text_field( $_POST["sms_provider_{$type}"] ?? 'hubtel' ) );
                update_option( "highest_sms_template_{$type}", sanitize_textarea_field( $_POST["sms_template_{$type}"] ?? '' ) );
            }
            update_option( 'highest_hubtel_client_id', sanitize_text_field( $_POST['hubtel_client_id'] ?? '' ) );
            update_option( 'highest_hubtel_client_secret', sanitize_text_field( $_POST['hubtel_client_secret'] ?? '' ) );
            update_option( 'highest_hubtel_sender_id', sanitize_text_field( $_POST['hubtel_sender_id'] ?? 'HIGHEST' ) );
            update_option( 'highest_bulksmsgh_api_key', sanitize_text_field( $_POST['bulksmsgh_api_key'] ?? '' ) );
            update_option( 'highest_bulksmsgh_sender_id', sanitize_text_field( $_POST['bulksmsgh_sender_id'] ?? '' ) );
            update_option( 'highest_nalo_api_key', sanitize_text_field( $_POST['nalo_api_key'] ?? '' ) );
            update_option( 'highest_nalo_sender_id', sanitize_text_field( $_POST['nalo_sender_id'] ?? '' ) );
            update_option( 'highest_moolre_vas_key', sanitize_text_field( $_POST['moolre_vas_key'] ?? '' ) );
            update_option( 'highest_moolre_sender_id', sanitize_text_field( $_POST['moolre_sender_id'] ?? '' ) );
            update_option( 'highest_payment_gateway', sanitize_text_field( $_POST['payment_gateway'] ?? 'paystack' ) );
            update_option( 'highest_moolre_api_user', sanitize_text_field( $_POST['moolre_api_user'] ?? '' ) );
            update_option( 'highest_moolre_pub_key', sanitize_text_field( $_POST['moolre_pub_key'] ?? '' ) );
            update_option( 'highest_moolre_account_number', sanitize_text_field( $_POST['moolre_account_number'] ?? '' ) );
            update_option( 'highest_moolre_webhook_secret', sanitize_text_field( $_POST['moolre_webhook_secret'] ?? '' ) );
            update_option( 'highest_sms_enabled', isset( $_POST['sms_enabled'] ) ? 'yes' : 'no' );
            update_option( 'highest_emergency_sms_enabled', isset( $_POST['emergency_sms_enabled'] ) ? 'yes' : 'no' );
            update_option( 'highest_emergency_prompt_enabled', isset( $_POST['emergency_prompt_enabled'] ) ? 'yes' : 'no' );  // new
            update_option( 'highest_emergency_sms_phone', sanitize_text_field( $_POST['emergency_sms_phone'] ?? '' ) );
            update_option( 'highest_public_announcement_enabled', isset( $_POST['public_announcement_enabled'] ) );
            update_option( 'highest_public_announcement_text', sanitize_textarea_field( $_POST['public_announcement_text'] ?? '' ) );
            update_option( 'highest_dk_balance_threshold', floatval( $_POST['highest_dk_balance_threshold'] ?? 0 ) );
            update_option( 'highest_low_balance_alert_enabled', isset( $_POST['low_balance_alert_enabled'] ) ? 'yes' : 'no' );
            // Referral system settings
            update_option( 'highest_enable_referral_system', isset( $_POST['enable_referral_system'] ) ? 'yes' : 'no' );
            update_option( 'highest_referral_bonus_limit_pct', intval( $_POST['referral_bonus_limit_pct'] ?? 20 ) );
            update_option( 'highest_require_referral_to_register', isset( $_POST['require_referral_to_register'] ) ? 'yes' : 'no' );
            update_option( 'highest_pause_agent_registration', isset( $_POST['pause_agent_registration'] ) ? 'yes' : 'no' );
            update_option( 'highest_agent_registration_approval', isset( $_POST['agent_registration_approval'] ) ? 'yes' : 'no' );
            update_option( 'highest_withdrawal_requires_approval', isset( $_POST['withdrawal_requires_approval'] ) ? 'yes' : 'no' );
            update_option( 'highest_enable_referral_maturation', isset( $_POST['enable_referral_maturation'] ) ? 'yes' : 'no' );
            update_option( 'highest_referral_maturation_threshold', floatval( $_POST['referral_maturation_threshold'] ?? 50 ) );
            // Estimated delivery & messages
            update_option( 'highest_estimated_delivery_minutes', intval( $_POST['estimated_delivery_minutes'] ?? 3 ) );
            update_option( 'highest_walkin_success_msg', sanitize_textarea_field( $_POST['walkin_success_msg'] ?? '' ) );
            update_option( 'highest_agent_success_msg', sanitize_textarea_field( $_POST['agent_success_msg'] ?? '' ) );
            update_option( 'highest_tracker_header_msg', sanitize_textarea_field( $_POST['tracker_header_msg'] ?? '' ) );
            // WAEC settings
            update_option( 'highest_waec_enabled', isset( $_POST['waec_enabled'] ) ? 'yes' : 'no' );
            update_option( 'highest_waec_bece_price', floatval( $_POST['waec_bece_price'] ?? 15 ) );
            update_option( 'highest_waec_bece_private_price', floatval( $_POST['waec_bece_private_price'] ?? 15 ) );
            update_option( 'highest_waec_wassce_price', floatval( $_POST['waec_wassce_price'] ?? 20 ) );
            update_option( 'highest_waec_novdec_price', floatval( $_POST['waec_novdec_price'] ?? 25 ) );
            update_option( 'highest_waec_sms_template', sanitize_textarea_field( $_POST['waec_sms_template'] ?? '' ) );
            echo '<div class="notice notice-success"><p>✅ Settings saved!</p></div>';
        }
        $pkey = get_option( 'highest_paystack_public_key', '' );
        $skey = get_option( 'highest_paystack_secret_key', '' );
        $dkey = get_option( 'highest_dakazina_api_key', '' );
        $dakazina_webhook = home_url( '/?hp_webhook=callback' );
        $disable_public = get_option( 'highest_disable_public_buy', false );
        $disable_agent  = get_option( 'highest_disable_agent_buy', false );
        $disable_all    = get_option( 'highest_disable_all_buy', false );
        $disable_marketplace_banner = get_option( 'highest_disable_marketplace_banner', false );
        $whatsapp        = get_option( 'highest_whatsapp_number', '0551907463' );
        $community       = get_option( 'highest_community_link', '' );
        $daily_bonus     = get_option( 'highest_daily_bonus_amount', 0.5 );
        $hubtel_client_id = get_option( 'highest_hubtel_client_id', '' );
        $hubtel_secret    = get_option( 'highest_hubtel_client_secret', '' );
        $hubtel_sender    = get_option( 'highest_hubtel_sender_id', 'HIGHEST' );
        $sms_enabled      = get_option( 'highest_sms_enabled', 'yes' ) === 'yes';
        $emergency_enabled = get_option( 'highest_emergency_sms_enabled', 'yes' ) === 'yes';
        $emergency_prompt_enabled = get_option( 'highest_emergency_prompt_enabled', 'yes' ) === 'yes'; // new
        $emergency_phone   = get_option( 'highest_emergency_sms_phone', '' );
        $public_announce_enabled = get_option( 'highest_public_announcement_enabled', false );
        $public_announce_text    = get_option( 'highest_public_announcement_text', '' );
        ?>
        <div class="wrap">
            <h1>HIGHEST DATA PLUG Settings</h1>
            <form method="post">
                <p>Paystack Public Key:</p>
                <input type="text" name="paystack_public" value="<?php echo esc_attr( $pkey ); ?>" style="width:100%;max-width:600px;"><br><br>
                <p>Paystack Secret Key:</p>
                <input type="password" name="paystack_secret" value="<?php echo esc_attr( $skey ); ?>" style="width:100%;max-width:600px;"><br><br>

                <h2>💳 Payment Gateway</h2>
                <label><input type="radio" name="payment_gateway" value="paystack" <?php checked( get_option( 'highest_payment_gateway', 'paystack' ), 'paystack' ); ?>> Paystack</label><br>
                <label><input type="radio" name="payment_gateway" value="moolre" <?php checked( get_option( 'highest_payment_gateway', 'paystack' ), 'moolre' ); ?>> Moolre</label>

                <div id="moolre-payment-settings" style="margin-top:16px;display:<?php echo get_option( 'highest_payment_gateway', 'paystack' ) === 'moolre' ? 'block' : 'none'; ?>;">
                    <h3>Moolre Payment Credentials</h3>
                    <p>API Username: <input type="text" name="moolre_api_user" value="<?php echo esc_attr( get_option('highest_moolre_api_user','') ); ?>" style="width:100%;max-width:600px;"></p>
                    <p>Public API Key: <input type="text" name="moolre_pub_key" value="<?php echo esc_attr( get_option('highest_moolre_pub_key','') ); ?>" style="width:100%;max-width:600px;"></p>
                    <p>Account Number: <input type="text" name="moolre_account_number" value="<?php echo esc_attr( get_option('highest_moolre_account_number','') ); ?>" style="width:100%;max-width:300px;"></p>
                    <p>Webhook Secret (optional): <input type="text" name="moolre_webhook_secret" value="<?php echo esc_attr( get_option('highest_moolre_webhook_secret','') ); ?>" style="width:100%;max-width:300px;"></p>
                    <p><strong>Webhook URL to give Moolre:</strong> <code><?php echo esc_url( home_url( '/wp-json/highest/v1/moolre-webhook' ) ); ?></code></p>
                </div>
                <script>
                (function() {
                    var radios = document.getElementsByName('payment_gateway');
                    for (var i = 0; i < radios.length; i++) {
                        radios[i].addEventListener('change', function() {
                            document.getElementById('moolre-payment-settings').style.display = this.value === 'moolre' ? 'block' : 'none';
                        });
                    }
                })();
                </script>
                <input type="text" name="dakazina_api_key" value="<?php echo esc_attr( $dkey ); ?>" style="width:100%;max-width:600px;"><br><br>
                <h2>📱 SMS Notification Settings</h2>
                <label><input type="checkbox" name="sms_enabled" <?php checked( $sms_enabled ); ?>> <strong>Master Enable SMS</strong> (global on/off – overrides everything below)</label><br><br>

                <?php
                $sms_message_types = array(
                    'order_received'     => 'Order Received (customer)',
                    'delivery_customer'  => 'Delivery Confirmation (customer)',
                    'delivery_agent'     => 'Delivery Confirmation (agent)',
                    'order_placed_agent' => 'Order Placed – Agent Notification',
                    'wallet_topup'       => 'Wallet Top-up (agent)',
                    'withdrawal_update'  => 'Withdrawal Approved/Rejected (agent)',
                    'manual_topup_agent' => 'Manual Top-up Approved (agent)',
                    'welcome_agent'      => 'New Agent Welcome',
                    'emergency_admin'    => 'Emergency Alerts (admin)',
                    'retry_sms'          => 'Retry SMS',
                    'notify_agent'       => 'Notify Agent (payment failure)',
                    'scheduled_campaign' => 'Scheduled Campaigns',
                    'bulk_agents'        => 'Bulk SMS to Agents',
                    'bulk_customers'     => 'Bulk SMS to Customers',
                );
                $sms_providers = array(
                    'hubtel'    => 'Hubtel',
                    'bulksmsgh' => 'BulkSMS Ghana',
                    'nalo'      => 'Nalo Solutions',
                    'moolre'    => 'Moolre SMS',
                );
                ?>
                <?php
                $default_templates = array(
                    'order_received'     => 'We have received your order for {package}. Your data will be delivered shortly. Track at: {tracking_url}',
                    'delivery_customer'  => 'Your data package ({package}) has been delivered successfully. Kindly allow sometime to reflect. Thank you for choosing.',
                    'delivery_agent'     => 'Order {package} to {phone} has been DELIVERED. Thank you.',
                    'order_placed_agent' => 'Order PLACED: {package} to {phone} has been submitted and is being processed.',
                    'wallet_topup'       => 'Your wallet has been credited with GH₵{amount}. New balance: GH₵{balance}',
                    'withdrawal_update'  => 'Your withdrawal request of GH₵{amount} has been approved. Funds will be sent to your account shortly.',
                    'manual_topup_agent' => 'Your manual top-up of GH₵{amount} has been approved. New wallet balance: GH₵{balance}',
                    'welcome_agent'      => 'Welcome {name}! Your HIGHEST DATA PLUG agent account is ready. Login at {login_url}',
                    'emergency_admin'    => 'EMERGENCY: Payment verified (ref: {ref}) but data delivery FAILED. Package: {package} to {phone}. Amount: GH₵{amount}. Agent ID: {agent_id}. Please check manually.',
                    'retry_sms'          => 'We noticed your purchase attempt was not completed. Please try again or contact support on {support}.',
                    'notify_agent'       => 'Customer {phone} attempted to buy {package} but the payment did not complete. Please assist them.',
                    'scheduled_campaign' => '{message}',
                    'bulk_agents'        => '{message}',
                    'bulk_customers'     => '{message}',
                );
                $placeholder_hints = array(
                    'order_received'     => 'Placeholders: {package}, {tracking_url}',
                    'delivery_customer'  => 'Placeholders: {package}',
                    'delivery_agent'     => 'Placeholders: {package}, {phone}',
                    'order_placed_agent' => 'Placeholders: {package}, {phone}',
                    'wallet_topup'       => 'Placeholders: {amount}, {balance}',
                    'withdrawal_update'  => 'Placeholders: {amount}',
                    'manual_topup_agent' => 'Placeholders: {amount}, {balance}',
                    'welcome_agent'      => 'Placeholders: {name}, {login_url}',
                    'emergency_admin'    => 'Placeholders: {ref}, {package}, {phone}, {amount}, {agent_id}',
                    'retry_sms'          => 'Placeholders: {support}',
                    'notify_agent'       => 'Placeholders: {phone}, {package}',
                    'scheduled_campaign' => 'The {message} placeholder contains the fully composed campaign message.',
                    'bulk_agents'        => 'The {message} placeholder contains the message typed in the Bulk SMS form.',
                    'bulk_customers'     => 'The {message} placeholder contains the message typed in the Bulk SMS to Customers form.',
                );
                ?>
                <table class="form-table" style="max-width:1000px;">
                    <thead>
                        <tr>
                            <th scope="col">Notification</th>
                            <th scope="col" style="width:60px;text-align:center;">Enable</th>
                            <th scope="col" style="width:180px;">Provider</th>
                            <th scope="col">Message Template</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php foreach ( $sms_message_types as $type => $label ) :
                            $current_template = get_option( "highest_sms_template_{$type}", $default_templates[ $type ] ?? '' );
                        ?>
                        <tr>
                            <td style="vertical-align:top;padding-top:12px;"><strong><?php echo esc_html( $label ); ?></strong></td>
                            <td style="text-align:center;vertical-align:top;padding-top:14px;">
                                <input type="checkbox" name="sms_enabled_<?php echo esc_attr( $type ); ?>" <?php checked( get_option( "highest_sms_enabled_{$type}", 'yes' ), 'yes' ); ?>>
                            </td>
                            <td style="vertical-align:top;padding-top:10px;">
                                <select name="sms_provider_<?php echo esc_attr( $type ); ?>" style="width:100%;">
                                    <?php foreach ( $sms_providers as $key => $name ) : ?>
                                    <option value="<?php echo esc_attr( $key ); ?>" <?php selected( get_option( "highest_sms_provider_{$type}", 'hubtel' ), $key ); ?>><?php echo esc_html( $name ); ?></option>
                                    <?php endforeach; ?>
                                </select>
                            </td>
                            <td style="vertical-align:top;">
                                <textarea name="sms_template_<?php echo esc_attr( $type ); ?>" rows="2" style="width:100%;max-width:440px;font-family:monospace;font-size:12px;"><?php echo esc_textarea( $current_template ); ?></textarea>
                                <p class="description" style="font-size:11px;margin-top:2px;"><?php echo esc_html( $placeholder_hints[ $type ] ?? '' ); ?></p>
                            </td>
                        </tr>
                        <?php endforeach; ?>
                    </tbody>
                </table>

                <h3 style="margin-top:24px;">🔷 Hubtel Credentials</h3>
                <p>Client ID: <input type="text" name="hubtel_client_id" value="<?php echo esc_attr( get_option('highest_hubtel_client_id','') ); ?>" style="width:100%;max-width:600px;"></p>
                <p>Client Secret: <input type="password" name="hubtel_client_secret" value="<?php echo esc_attr( get_option('highest_hubtel_client_secret','') ); ?>" style="width:100%;max-width:600px;"></p>
                <p>Sender ID: <input type="text" name="hubtel_sender_id" value="<?php echo esc_attr( get_option('highest_hubtel_sender_id','HIGHEST') ); ?>" style="width:100%;max-width:600px;"></p>

                <h3 style="margin-top:24px;">🟢 BulkSMS Ghana Credentials</h3>
                <p>API Key: <input type="text" name="bulksmsgh_api_key" value="<?php echo esc_attr( get_option('highest_bulksmsgh_api_key','') ); ?>" style="width:100%;max-width:600px;"></p>
                <p>Sender ID: <input type="text" name="bulksmsgh_sender_id" value="<?php echo esc_attr( get_option('highest_bulksmsgh_sender_id','') ); ?>" maxlength="11" style="width:100%;max-width:600px;"></p>
                <p class="description">Sender ID must not exceed 11 characters.</p>

                <h3 style="margin-top:24px;">🔵 Nalo Solutions Credentials</h3>
                <p>API Key: <input type="text" name="nalo_api_key" value="<?php echo esc_attr( get_option('highest_nalo_api_key','') ); ?>" style="width:100%;max-width:600px;"></p>
                <p>Sender ID: <input type="text" name="nalo_sender_id" value="<?php echo esc_attr( get_option('highest_nalo_sender_id','') ); ?>" maxlength="11" style="width:100%;max-width:600px;"></p>
                <p class="description">Must be alphanumeric, max 11 characters. No spaces.</p>

                <h3 style="margin-top:24px;">🟠 Moolre SMS Credentials</h3>
                <p>VAS Key: <input type="text" name="moolre_vas_key" value="<?php echo esc_attr( get_option('highest_moolre_vas_key','') ); ?>" style="width:100%;max-width:600px;"></p>
                <p>Sender ID: <input type="text" name="moolre_sender_id" value="<?php echo esc_attr( get_option('highest_moolre_sender_id','') ); ?>" maxlength="11" style="width:100%;max-width:600px;"></p>
                <p class="description">Sender ID must not exceed 11 characters. No spaces.</p>
                <p><strong>Admin phone number that receives all emergency alerts:</strong></p>
                <input type="text" name="emergency_sms_phone" value="<?php echo esc_attr( $emergency_phone ); ?>" style="width:100%;max-width:300px; margin-bottom:15px;">

                <label><input type="checkbox" name="emergency_sms_enabled" <?php checked( $emergency_enabled ); ?>> <strong>Automatic delivery‑failure SMS</strong><br>
                <span style="margin-left:20px; font-size:13px; color:#666;">Sends SMS to admin instantly when DataKazina fails to deliver after a successful payment.</span></label><br><br>

                <label><input type="checkbox" name="emergency_prompt_enabled" <?php checked( $emergency_prompt_enabled ); ?>> <strong>Agent SOS prompt on order failure</strong><br>
                <span style="margin-left:20px; font-size:13px; color:#666;">After an agent’s order fails to create, show a confirmation dialog asking if they want to notify admin via SMS.</span></label>

                <h2>💰 DataKazina Balance Alert</h2>
                <p>Alert Threshold (GH₵): <input type="number" name="highest_dk_balance_threshold" value="<?php echo esc_attr( get_option( 'highest_dk_balance_threshold', '' ) ); ?>" step="0.01" style="width:150px;"></p>
                <label><input type="checkbox" name="low_balance_alert_enabled" <?php checked( get_option( 'highest_low_balance_alert_enabled', 'no' ), 'yes' ); ?>> Enable low-balance SMS alert (sent to Emergency SMS phone above)</label>

                <h2>🔗 Agent Referral System</h2>
                <label><input type="checkbox" name="enable_referral_system" <?php checked( get_option( 'highest_enable_referral_system', 'no' ), 'yes' ); ?>> Enable Referral System</label><br><br>
                <p>Maximum Referral Bonus (% of sub-agent's profit): <input type="number" name="referral_bonus_limit_pct" value="<?php echo esc_attr( get_option( 'highest_referral_bonus_limit_pct', 20 ) ); ?>" step="1" min="0" max="100" style="width:100px;">%</p>
                <p class="description" style="color:#666;font-size:13px;">Agents cannot set a bonus higher than this percentage.</p>

                <h3 style="margin-top:20px;">🔐 Agent Registration Control</h3>
                <label><input type="checkbox" name="require_referral_to_register" <?php checked( get_option( 'highest_require_referral_to_register', 'no' ), 'yes' ); ?>> Require referral link to register new agents</label><br><br>
                <label><input type="checkbox" name="pause_agent_registration" <?php checked( get_option( 'highest_pause_agent_registration', 'no' ), 'yes' ); ?>> Pause ALL new agent registrations</label><br><br>
                <label><input type="checkbox" name="agent_registration_approval" <?php checked( get_option( 'highest_agent_registration_approval', 'no' ), 'yes' ); ?>> New agents must be approved by admin before they can access the portal</label>

                <h3 style="margin-top:20px;">🔒 Referral Bonus Maturation</h3>
                <label><input type="checkbox" name="enable_referral_maturation" <?php checked( get_option( 'highest_enable_referral_maturation', 'yes' ), 'yes' ); ?>> Enable bonus maturation (lock bonuses until threshold is reached)</label><br><br>
                <p>Claim Threshold (GH₵) – total locked must reach this before agent can claim:
                    <input type="number" name="referral_maturation_threshold" value="<?php echo esc_attr( get_option( 'highest_referral_maturation_threshold', 50 ) ); ?>" step="0.01" min="0" style="width:150px;">
                </p>
                <p class="description" style="color:#666;font-size:13px;">When enabled, referral bonuses are held in a locked pot. The agent can move them to their withdrawable profit only once the total locked amount reaches this threshold. When disabled, bonuses are credited directly (original behaviour).</p>

                <h2>💸 Withdrawal Settings</h2>
                <label><input type="checkbox" name="withdrawal_requires_approval" <?php checked( get_option( 'highest_withdrawal_requires_approval', 'yes' ), 'yes' ); ?>> Require admin approval for automatic (MoMo) withdrawals</label>
                <p class="description">When disabled, automatic (MoMo) withdrawals are sent instantly and marked completed. <strong>Manual (bank transfer) withdrawals always require admin approval.</strong></p>

                <h2>📣 Public Buy Form Announcement</h2>
                <label><input type="checkbox" name="public_announcement_enabled" <?php checked( $public_announce_enabled ); ?>> Show announcement on the public buy page</label><br><br>
                <p>Announcement Text (HTML allowed):</p>
                <textarea name="public_announcement_text" rows="4" style="width:100%;max-width:600px;"><?php echo esc_textarea( $public_announce_text ); ?></textarea>
                <h2>🔧 Feature Toggles</h2>
                <label><input type="checkbox" name="disable_public_buy" <?php checked( $disable_public ); ?>> Disable Public Buy</label><br><br>
                <label><input type="checkbox" name="disable_agent_buy" <?php checked( $disable_agent ); ?>> Disable Agent Buy</label><br><br>
                <label><input type="checkbox" name="disable_all_buy" <?php checked( $disable_all ); ?>> Disable EVERY Buy Button</label><br><br>
                <h2>🛒 Marketplace Banner</h2>
                <label><input type="checkbox" name="disable_marketplace_banner" <?php checked( $disable_marketplace_banner ); ?>> Disable Marketplace Banner</label><br><br>
                <label><input type="checkbox" name="show_whatsapp_banner" <?php checked( get_option( 'highest_show_whatsapp_banner', 'yes' ), 'yes' ); ?>> Show WhatsApp Community Banner on public page</label><br><br>
                <h2>🎁 Gamification</h2>
                <p>Daily Login Bonus (GH₵):</p>
                <input type="number" name="daily_bonus_amount" step="0.01" value="<?php echo esc_attr( $daily_bonus ); ?>"><br><br>
                <h2>⏱️ Estimated Delivery &amp; Messages</h2>
                <p>Estimated delivery time (minutes): <input type="number" name="estimated_delivery_minutes" value="<?php echo esc_attr( get_option( 'highest_estimated_delivery_minutes', 3 ) ); ?>" step="1" min="1" max="60" style="width:100px;"> min</p>
                <p class="description" style="color:#666;font-size:13px;margin-bottom:12px;">Use <code>{minutes}</code> as a placeholder in the messages below.</p>
                <p><strong>Walk-in Success Message</strong> (shown after public purchase):</p>
                <textarea name="walkin_success_msg" rows="2" style="width:100%;max-width:600px;"><?php echo esc_textarea( get_option( 'highest_walkin_success_msg', '✅ Order placed successfully! Delivery usually takes about {minutes} minute(s).' ) ); ?></textarea><br><br>
                <p><strong>Agent Success Message</strong> (shown after agent buys for a customer):</p>
                <textarea name="agent_success_msg" rows="2" style="width:100%;max-width:600px;"><?php echo esc_textarea( get_option( 'highest_agent_success_msg', '✅ Data purchased successfully! Delivery usually takes about {minutes} minute(s).' ) ); ?></textarea><br><br>
                <p><strong>Public Tracker Header Message</strong>:</p>
                <textarea name="tracker_header_msg" rows="2" style="width:100%;max-width:600px;"><?php echo esc_textarea( get_option( 'highest_tracker_header_msg', 'Data delivery usually takes about {minutes} minute(s).' ) ); ?></textarea>
                <p class="description" style="color:#666;font-size:13px;">Use <code>{minutes}</code> as placeholder.</p>
                <h2>📱 Support</h2>
                <h2>📞 USSD Settings</h2>
                <p>Agent prefix: <input type="text" name="ussd_agent_prefix" value="<?php echo esc_attr( get_option( 'highest_ussd_agent_prefix', '110' ) ); ?>" style="width:100px;"></p>
                <p class="description">Dial *203*[PREFIX]*AGENTID# to reach an agent. (e.g. with default prefix 110, dial *203*110*53#). Leave empty if using a direct shortcode like *919*909#.</p>
                <p style="margin-top:12px;">Full shortcode: <input type="text" name="ussd_shortcode" value="<?php echo esc_attr( get_option( 'highest_ussd_shortcode', '*919*909*' ) ); ?>" style="width:200px;"></p>
                <p class="description">Shown to agents in their portal as their personal dial code. Do not include the agent ID – it is appended automatically. Example: <code>*919*909*</code></p>
                <p style="margin-top:12px;"><strong>USSD Payment Gateway</strong></p>
                <select name="ussd_payment_gateway">
                    <option value="" <?php selected( get_option( 'highest_ussd_payment_gateway', '' ), '' ); ?>>Same as website</option>
                    <option value="paystack" <?php selected( get_option( 'highest_ussd_payment_gateway', '' ), 'paystack' ); ?>>Paystack</option>
                    <option value="moolre" <?php selected( get_option( 'highest_ussd_payment_gateway', '' ), 'moolre' ); ?>>Moolre</option>
                </select>
                <p class="description">USSD will always use this gateway. If "Same as website" is selected, it follows the main payment gateway setting.</p>
                <h2>📱 Support</h2>
                <p>WhatsApp Number:</p>
                <input type="text" name="whatsapp_number" value="<?php echo esc_attr( $whatsapp ); ?>"><br><br>
                <p>Community Link:</p>
                <input type="url" name="community_link" value="<?php echo esc_attr( $community ); ?>"><br><br>

                <!-- Save Tier Prices inline with main form -->
                <?php
                // Handle save within main form
                if ( isset( $_POST['highest_save'] ) ) {
                    // Wholesale toggle
                    update_option( 'highest_enable_wholesale_pricing', isset( $_POST['enable_wholesale_pricing'] ) ? 'yes' : 'no' );
                    // Tier upgrade toggle + fees
                    update_option( 'highest_enable_tier_upgrade', isset( $_POST['enable_tier_upgrade'] ) ? 'yes' : 'no' );
                    $upgrade_fees = array();
                    foreach ( $_POST['tier_upgrade_fee'] ?? array() as $tier => $fee ) {
                        $fee = floatval( $fee );
                        if ( $fee > 0 ) $upgrade_fees[ sanitize_text_field( $tier ) ] = $fee;
                    }
                    update_option( 'highest_tier_upgrade_fees', $upgrade_fees );
                }
                ?>

                <h2 style="margin-top:30px;">🏷️ Wholesale Pricing for Sub-Agents</h2>
                <label>
                    <input type="checkbox" name="enable_wholesale_pricing" <?php checked( get_option( 'highest_enable_wholesale_pricing', 'no' ), 'yes' ); ?>>
                    Enable wholesale pricing
                </label>
                <p class="description">When enabled, parent agents can set specific wholesale prices for their sub-agents. Sub-agents pay those prices instead of tier cost, and the parent earns the margin.</p>

                <h2 style="margin-top:20px;">⬆️ Agent Tier Self-Upgrade</h2>
                <label>
                    <input type="checkbox" name="enable_tier_upgrade" <?php checked( get_option( 'highest_enable_tier_upgrade', 'no' ), 'yes' ); ?>>
                    Enable agent self-upgrade
                </label>
                <p class="description">When enabled, agents can pay to move to a higher tier.</p>
                <h3>Upgrade Fees (GH₵)</h3>
                <p class="description">Set the fee for each tier upgrade. Leave blank or 0 to disable that upgrade path.</p>
                <?php
                $upgrade_fees_display = get_option( 'highest_tier_upgrade_fees', array() );
                $tiers_for_fees = get_option( 'highest_agent_tiers', array( 'standard' => 'Standard' ) );
                foreach ( $tiers_for_fees as $target_tier => $target_name ) :
                    if ( $target_tier === 'standard' ) continue;
                ?>
                <p>Upgrade to <strong><?php echo esc_html( $target_name ); ?></strong>:
                    <input type="number" name="tier_upgrade_fee[<?php echo esc_attr( $target_tier ); ?>]" value="<?php echo esc_attr( $upgrade_fees_display[ $target_tier ] ?? '' ); ?>" step="0.01" style="width:100px;" placeholder="GH₵"></p>
                <?php endforeach; ?>

                <h2 style="margin-top:30px;">📝 WAEC Scratch Cards</h2>
                <p>Enable WAEC Sales:
                    <input type="checkbox" name="waec_enabled" <?php checked( get_option( 'highest_waec_enabled', 'no' ), 'yes' ); ?>>
                </p>
                <p>BECE (School) Price (GH₵): <input type="number" name="waec_bece_price" value="<?php echo esc_attr( get_option( 'highest_waec_bece_price', 15 ) ); ?>" step="0.01" style="width:120px;"></p>
                <p>BECE Private Price (GH₵): <input type="number" name="waec_bece_private_price" value="<?php echo esc_attr( get_option( 'highest_waec_bece_private_price', 15 ) ); ?>" step="0.01" style="width:120px;"></p>
                <p>WASSCE (School) Price (GH₵): <input type="number" name="waec_wassce_price" value="<?php echo esc_attr( get_option( 'highest_waec_wassce_price', 20 ) ); ?>" step="0.01" style="width:120px;"></p>
                <p>WASSCE Private (Nov/Dec) Price (GH₵): <input type="number" name="waec_novdec_price" value="<?php echo esc_attr( get_option( 'highest_waec_novdec_price', 25 ) ); ?>" step="0.01" style="width:120px;"></p>
                <p>Delivery SMS Template:</p>
                <textarea name="waec_sms_template" rows="3" style="width:100%;max-width:600px;"><?php echo esc_textarea( get_option( 'highest_waec_sms_template', 'Your {exam} checker is ready. Serial: {serial}, PIN: {pin}. Visit waecdirect.org to check results.' ) ); ?></textarea>

                <input type="submit" name="highest_save" value="Save Settings" class="button button-primary">
            </form>
            <?php
            if ( isset( $_POST['save_tiers'] ) && check_admin_referer( 'hdp_tiers' ) ) {
                $new_tiers = array();
                $tier_names = $_POST['tier_names'] ?? array();
                $tier_slugs = $_POST['tier_slugs'] ?? array();
                $new_tiers['standard'] = 'Standard';
                foreach ( $tier_slugs as $i => $slug ) {
                    $slug = sanitize_title( $slug );
                    $name = sanitize_text_field( $tier_names[ $i ] ?? '' );
                    if ( ! empty( $slug ) && ! empty( $name ) && $slug !== 'standard' ) {
                        $new_tiers[ $slug ] = $name;
                    }
                }
                update_option( 'highest_agent_tiers', $new_tiers );
                echo '<div class="notice notice-success"><p>✅ Tiers updated.</p></div>';
            }
            if ( isset( $_POST['save_tier_prices'] ) && check_admin_referer( 'hdp_tier_prices' ) ) {
                $prices = $_POST['tier_prices'] ?? array();
                $sanitized = array();
                foreach ( $prices as $tier => $package_prices ) {
                    foreach ( $package_prices as $pkg => $price ) {
                        $sanitized[ $tier ][ $pkg ] = floatval( $price );
                    }
                }
                update_option( 'highest_tier_prices', $sanitized );
                echo '<div class="notice notice-success"><p>✅ Tier prices updated.</p></div>';
            }
            $tiers = get_option( 'highest_agent_tiers', array( 'standard' => 'Standard' ) );
            $packages = array_keys( $this->get_default_agent_prices() );
            $saved_prices = get_option( 'highest_tier_prices', array() );
            ?>

            <form method="post">
                <?php wp_nonce_field( 'hdp_tiers' ); ?>
                <h3>Agent Tiers</h3>
                <table class="form-table" id="tier-names-table">
                    <thead><tr><th>Slug</th><th>Display Name</th><th>Action</th></tr></thead>
                    <tbody>
                        <?php foreach ( $tiers as $slug => $name ) : ?>
                        <tr>
                            <td><code><?php echo esc_html( $slug ); ?></code></td>
                            <td>
                                <?php if ( $slug === 'standard' ) : ?>
                                    <strong>Standard</strong> <em>(locked)</em>
                                <?php else : ?>
                                    <input type="text" name="tier_names[]" value="<?php echo esc_attr( $name ); ?>" class="regular-text">
                                    <input type="hidden" name="tier_slugs[]" value="<?php echo esc_attr( $slug ); ?>">
                                <?php endif; ?>
                            </td>
                            <td>
                                <?php if ( $slug !== 'standard' ) : ?>
                                    <a href="#" class="button button-small" onclick="this.closest('tr').remove(); return false;">Remove</a>
                                <?php endif; ?>
                            </td>
                        </tr>
                        <?php endforeach; ?>
                    </tbody>
                </table>
                <button type="button" class="button" id="add-tier-row">+ Add New Tier</button>
                <p class="submit"><button type="submit" name="save_tiers" class="button button-primary">Save Tiers</button></p>
            </form>
            <script>
            jQuery(function($){
                $('#add-tier-row').click(function(){
                    var slug = prompt('Enter a slug for the new tier (e.g. silver):');
                    if (!slug) return;
                    var name = prompt('Enter display name (e.g. Silver):');
                    if (!name) return;
                    var row = '<tr><td><code>'+slug+'</code></td><td><input type="text" name="tier_names[]" value="'+name+'" class="regular-text"><input type="hidden" name="tier_slugs[]" value="'+slug+'"></td><td><a href="#" class="button button-small" onclick="this.closest(\'tr\').remove(); return false;">Remove</a></td></tr>';
                    $('#tier-names-table tbody').append(row);
                });
            });
            </script>

            <form method="post" style="margin-top:40px;">
                <?php wp_nonce_field( 'hdp_tier_prices' ); ?>
                <h3>Package Costs per Tier</h3>
                <table class="wp-list-table widefat fixed striped">
                    <thead>
                        <tr>
                            <th>Package</th>
                            <?php foreach ( $tiers as $slug => $name ) : ?>
                                <th><?php echo esc_html( $name ); ?> (GH₵)</th>
                            <?php endforeach; ?>
                        </tr>
                    </thead>
                    <tbody>
                        <?php foreach ( $packages as $pkg ) : ?>
                        <tr>
                            <td><?php echo esc_html( $pkg ); ?></td>
                            <?php foreach ( $tiers as $slug => $name ) :
                                $current = $saved_prices[ $slug ][ $pkg ] ?? $this->get_default_agent_prices()[ $pkg ];
                            ?>
                                <td><input type="number" name="tier_prices[<?php echo esc_attr( $slug ); ?>][<?php echo esc_attr( $pkg ); ?>]" value="<?php echo number_format( $current, 2 ); ?>" step="0.01" style="width:100px;"></td>
                            <?php endforeach; ?>
                        </tr>
                        <?php endforeach; ?>
                    </tbody>
                </table>
                <p class="submit"><button type="submit" name="save_tier_prices" class="button button-primary">Save Tier Prices</button></p>
            </form>

            <hr>
            <h2>Webhook URLs</h2>
            <p>DataKazina: <code><?php echo esc_html( $dakazina_webhook ); ?></code></p>
            <p>Paystack: <code><?php echo esc_url( home_url( '/wp-json/highest/v1/paystack-webhook' ) ); ?></code></p>
        </div>
        <?php
    }

    // ─── Withdrawals Page (unchanged from v4.2.1) ──
    public function withdrawals_page() {
        $all_requests = [];
        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        foreach ( $agents as $agent ) {
            $requests = get_user_meta( $agent->ID, 'highest_withdraw_requests', true ) ?: [];
            foreach ( $requests as $req ) {
                if ( ! is_array( $req ) ) continue;
                $req['agent_id']   = $agent->ID;
                $req['agent_name'] = $agent->display_name;
                $all_requests[]    = $req;
            }
        }
        usort( $all_requests, function( $a, $b ) {
            return strtotime( $b['date'] ?? '1970-01-01' ) - strtotime( $a['date'] ?? '1970-01-01' );
        });
        $total_approved = 0;
        foreach ( $all_requests as $req ) {
            if ( isset( $req['status'] ) && $req['status'] === 'approved' ) {
                $total_approved += (float) ( $req['amount'] ?? 0 );
            }
        }
        if ( isset( $_GET['approved'] ) ) echo '<div class="notice notice-success is-dismissible"><p>✅ Withdrawal approved successfully.</p></div>';
        if ( isset( $_GET['rejected'] ) ) echo '<div class="notice notice-info is-dismissible"><p>❌ Withdrawal rejected.</p></div>';
        ?>
        <div class="wrap">
            <h1>💰 Withdrawal Requests Management</h1>
            <?php if ( $total_approved > 0 ) : ?>
                <div class="notice notice-info inline"><p><strong>Total Approved Withdrawals (all time):</strong> GH₵ <?php echo number_format( $total_approved, 2 ); ?></p></div>
            <?php endif; ?>
            <table class="wp-list-table widefat fixed striped" style="margin-top:20px;">
                <thead>
                    <tr><th>Agent</th><th>Date</th><th>Gross (GH₵)</th><th>Method</th><th>Fee</th><th>Net</th><th>Account / Bank</th><th>Status</th><th>Actions</th></tr>
                </thead>
                <tbody>
                    <?php if ( empty( $all_requests ) ) : ?>
                        <tr><td colspan="6" style="text-align:center; padding:40px;">No withdrawal requests yet.</td></tr>
                    <?php else : ?>
                        <?php foreach ( $all_requests as $req ) : 
                            $status = $req['status'] ?? 'pending_approval';
                            $status_label = ucfirst( str_replace( '_', ' ', $status ) );
                            $status_bg = '';
                            if ( $status === 'approved' ) $status_bg = 'background:#10b981;color:white;';
                            elseif ( $status === 'rejected' ) $status_bg = 'background:#ef4444;color:white;';
                            else $status_bg = 'background:#eab308;color:#1f2937;';
                            ?>
                            <?php
                                $req_method = $req['method'] ?? 'manual';
                                $req_fee    = (float) ( $req['fee'] ?? 0 );
                                $req_net    = (float) ( $req['net_amount'] ?? $req['amount'] ?? 0 );
                                $xfer_code  = $req['transfer_code'] ?? '';
                            ?>
                            <tr>
                                <td><strong><?php echo esc_html( $req['agent_name'] ); ?></strong><br><small>ID: <?php echo (int) $req['agent_id']; ?></small></td>
                                <td><?php echo esc_html( $req['date'] ?? '—' ); ?></td>
                                <td><strong>GH₵ <?php echo number_format( (float) ( $req['amount'] ?? 0 ), 2 ); ?></strong></td>
                                <td><span style="padding:2px 8px;border-radius:4px;font-size:12px;font-weight:bold;background:<?php echo $req_method === 'auto' ? '#6366f1' : '#64748b'; ?>;color:#fff;"><?php echo esc_html( strtoupper( $req_method ) ); ?></span></td>
                                <td><?php echo $req_fee > 0 ? 'GH₵ ' . number_format( $req_fee, 2 ) : '—'; ?></td>
                                <td><strong>GH₵ <?php echo number_format( $req_net, 2 ); ?></strong></td>
                                <td><?php
                                    if ( $req_method === 'auto' ) {
                                        $disp_provider = strtoupper( $req['provider'] ?? get_user_meta( $req['agent_id'], 'highest_momo_provider', true ) );
                                        $disp_phone    = $req['phone'] ?? get_user_meta( $req['agent_id'], 'highest_momo_phone', true );
                                        echo esc_html( $disp_provider . ' MoMo · ' . $disp_phone );
                                    } else {
                                        echo esc_html( $req['account'] ?? '—' );
                                    }
                                    if ( $xfer_code ) echo '<br><small style="color:#10b981;">Transfer: ' . esc_html( $xfer_code ) . '</small>';
                                ?></td>
                                <td><span style="display:inline-block; padding:4px 12px; border-radius:9999px; font-size:13px; <?php echo $status_bg; ?>"><?php echo esc_html( $status_label ); ?></span></td>
                                <td>
                                    <?php if ( $status === 'pending_approval' ) : ?>
                                        <form method="post" style="display:inline-block; margin-right:5px;"><input type="hidden" name="user_id" value="<?php echo (int) $req['agent_id']; ?>"><input type="hidden" name="request_id" value="<?php echo esc_attr( $req['id'] ?? '' ); ?>"><button type="submit" name="approve_withdrawal" class="button button-primary" style="min-width:70px;">Approve</button></form>
                                        <form method="post" style="display:inline-block;"><input type="hidden" name="user_id" value="<?php echo (int) $req['agent_id']; ?>"><input type="hidden" name="request_id" value="<?php echo esc_attr( $req['id'] ?? '' ); ?>"><button type="submit" name="reject_withdrawal" class="button button-secondary" onclick="return confirm('Reject this request?');">Reject</button></form>
                                    <?php else : ?>
                                        <span class="description">– Processed –</span>
                                    <?php endif; ?>
                                 </div>
                            </tr>
                        <?php endforeach; ?>
                    <?php endif; ?>
                </tbody>
            </table>
        </div>
        <?php
    }

    public function handle_withdrawal_approval() {
        if ( ! current_user_can( 'manage_options' ) ) return;
        if ( isset( $_POST['approve_withdrawal'] ) ) {
            $user_id = intval( $_POST['user_id'] );
            $request_id = sanitize_text_field( $_POST['request_id'] );
            $requests = get_user_meta( $user_id, 'highest_withdraw_requests', true ) ?: array();
            $amount = 0;
            foreach ( $requests as &$req ) {
                if ( isset( $req['id'] ) && $req['id'] === $request_id ) {
                    $req['status'] = 'approved';
                    $req['approved_at'] = current_time( 'mysql' );
                    $amount = (float) ( $req['amount'] ?? 0 );
                    // Auto-transfer via Paystack if method is auto
                    if ( ( $req['method'] ?? 'manual' ) === 'auto' ) {
                        $provider   = $req['provider'] ?? get_user_meta( $user_id, 'highest_momo_provider', true );
                        $phone      = $req['phone']    ?? get_user_meta( $user_id, 'highest_momo_phone', true );
                        $net_amount = $req['net_amount'] ?? $req['amount'];
                        $transfer_result = $this->process_auto_withdrawal( $user_id, $provider, $phone, $net_amount );
                        if ( ! is_wp_error( $transfer_result ) ) {
                            $req['status']        = 'completed';
                            $req['transfer_code'] = $transfer_result;
                        } else {
                            error_log( 'Paystack auto withdrawal failed for user ' . $user_id . ': ' . $transfer_result->get_error_message() );
                            $req['status'] = 'approved_transfer_failed';
                        }
                    }
                    break;
                }
            }
            update_user_meta( $user_id, 'highest_withdraw_requests', $requests );
            if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) ) {
                $agent_phone = $this->get_agent_phone( $user_id );
                if ( ! empty( $agent_phone ) && $amount > 0 ) {
                    $this->send_templated_sms( $agent_phone, 'withdrawal_update', array(
                        'amount' => number_format( $amount, 2 ),
                    ) );
                }
            }
            wp_redirect( admin_url( 'admin.php?page=highest-withdrawals&approved=1' ) );
            exit;
        }
        if ( isset( $_POST['reject_withdrawal'] ) ) {
            $user_id = intval( $_POST['user_id'] );
            $request_id = sanitize_text_field( $_POST['request_id'] );
            $requests = get_user_meta( $user_id, 'highest_withdraw_requests', true ) ?: array();
            $rejected_amount = 0;
            foreach ( $requests as &$req ) {
                if ( isset( $req['id'] ) && $req['id'] === $request_id ) {
                    $req['status'] = 'rejected';
                    $req['rejected_at'] = current_time( 'mysql' );
                    $rejected_amount = (float) ( $req['amount'] ?? 0 );
                    break;
                }
            }
            update_user_meta( $user_id, 'highest_withdraw_requests', $requests );
            if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) && $rejected_amount > 0 ) {
                $agent_phone = $this->get_agent_phone( $user_id );
                if ( ! empty( $agent_phone ) ) {
                    $this->send_templated_sms( $agent_phone, 'withdrawal_update', array(
                        'amount' => number_format( $rejected_amount, 2 ),
                    ) );
                }
            }
            wp_redirect( admin_url( 'admin.php?page=highest-withdrawals&rejected=1' ) );
            exit;
        }
    }

    private function process_auto_withdrawal( $user_id, $provider, $phone, $amount, $name = '' ) {
        $secret = get_option( 'highest_paystack_secret_key', '' );
        if ( empty( $secret ) ) return new WP_Error( 'no_key', 'Paystack secret key missing.' );

        $provider = strtolower( trim( $provider ) );

        // Paystack Ghana mobile money bank_code values.
        // Note: Vodafone Cash rebranded to Telecel Cash but Paystack still uses 'VOD'.
        $provider_codes = array(
            'mtn'        => 'MTN',   // MTN Mobile Money
            'telecel'    => 'VOD',   // Telecel Cash (formerly Vodafone Cash)
            'vodafone'   => 'VOD',   // Legacy alias
            'airteltigo' => 'ATL',   // AirtelTigo Money
        );

        if ( ! isset( $provider_codes[ $provider ] ) ) {
            return new WP_Error( 'invalid_provider', 'Unknown MoMo provider: ' . $provider );
        }
        $bank_code = $provider_codes[ $provider ];

        $recipient_name = ! empty( $name ) ? $name
            : ( get_user_meta( $user_id, 'highest_account_name', true )
                ?: get_user_by( 'id', $user_id )->display_name );

        // Step 1: Create transfer recipient
        $recipient_resp = wp_remote_post( 'https://api.paystack.co/transferrecipient', array(
            'headers' => array(
                'Authorization' => 'Bearer ' . $secret,
                'Content-Type'  => 'application/json',
            ),
            'body'    => wp_json_encode( array(
                'type'           => 'mobile_money',
                'name'           => $recipient_name,
                'account_number' => $phone,
                'bank_code'      => $bank_code,
                'currency'       => 'GHS',
            ) ),
            'timeout' => 30,
        ) );
        if ( is_wp_error( $recipient_resp ) ) return $recipient_resp;
        $recipient_body = json_decode( wp_remote_retrieve_body( $recipient_resp ), true );
        if ( empty( $recipient_body['status'] ) || ! $recipient_body['status'] ) {
            return new WP_Error( 'recipient_fail', $recipient_body['message'] ?? 'Failed to create transfer recipient.' );
        }
        $recipient_code = $recipient_body['data']['recipient_code'];

        // Step 2: Initiate transfer (amount in pesewas)
        $net_pesewas = intval( round( (float) $amount * 100 ) );
        $transfer_resp = wp_remote_post( 'https://api.paystack.co/transfer', array(
            'headers' => array(
                'Authorization' => 'Bearer ' . $secret,
                'Content-Type'  => 'application/json',
            ),
            'body'    => wp_json_encode( array(
                'source'    => 'balance',
                'amount'    => $net_pesewas,
                'recipient' => $recipient_code,
                'reason'    => 'Agent payout – HIGHEST DATA PLUG #' . $user_id,
                'currency'  => 'GHS',
            ) ),
            'timeout' => 30,
        ) );
        if ( is_wp_error( $transfer_resp ) ) return $transfer_resp;
        $transfer_body = json_decode( wp_remote_retrieve_body( $transfer_resp ), true );
        if ( empty( $transfer_body['status'] ) || ! $transfer_body['status'] ) {
            return new WP_Error( 'transfer_fail', $transfer_body['message'] ?? 'Paystack transfer initiation failed.' );
        }
        return $transfer_body['data']['transfer_code'];
    }

    public function handle_force_delivered() {
        if ( ! current_user_can( 'manage_options' ) ) return;
        if ( isset( $_POST['force_delivered_ref'] ) && isset( $_POST['force_delivered_order_id'] ) ) {
            check_admin_referer( 'hdp_force_delivered_' . intval( $_POST['force_delivered_order_id'] ), '_wpnonce_force_delivered' );
            $ref = sanitize_text_field( $_POST['force_delivered_ref'] );
            $this->update_order_status_by_ref( $ref, 'DELIVERED', array( 'api_response' => '{"note":"Admin Manually Forced Status to DELIVERED"}' ) );
            wp_redirect( admin_url( 'admin.php?page=highest-orders-payments&forced=1' ) );
            exit;
        }
    }

    public function maybe_add_balance_after_column() {
        global $wpdb;
        $table = $wpdb->prefix . 'highest_topup_history';
        $row = $wpdb->get_results( "SHOW COLUMNS FROM {$table} LIKE 'balance_after'" );
        if ( empty( $row ) ) {
            $wpdb->query( "ALTER TABLE {$table} ADD COLUMN balance_after DECIMAL(10,2) DEFAULT 0 AFTER amount" );
        }
    }

    private function get_delivery_message( $option_key, $default ) {
        $msg  = get_option( $option_key, $default );
        $mins = intval( get_option( 'highest_estimated_delivery_minutes', 3 ) );
        return str_replace( '{minutes}', $mins, $msg );
    }

        public function handle_refund_wallet() {
        if ( ! current_user_can( 'manage_options' ) || ! isset( $_POST['refund_wallet_ref'] ) || ! isset( $_POST['refund_wallet_order_id'] ) ) return;
        check_admin_referer( 'hdp_refund_wallet_' . intval( $_POST['refund_wallet_order_id'] ), '_wpnonce_refund_wallet' );
        $ref = sanitize_text_field( $_POST['refund_wallet_ref'] );
        global $wpdb;
        $order = $wpdb->get_row( $wpdb->prepare(
            "SELECT * FROM {$wpdb->prefix}highest_orders WHERE ref = %s AND status = 'FAILED' LIMIT 1", $ref
        ) );
        if ( ! $order || empty( $order->user_id ) || $order->agent_cost <= 0 ) {
            wp_redirect( admin_url( 'admin.php?page=highest-orders-payments&refund=error' ) );
            exit;
        }
        // Credit agent's wallet
        $wallet = (float) get_user_meta( $order->user_id, 'highest_wallet', true );
        update_user_meta( $order->user_id, 'highest_wallet', $wallet + $order->agent_cost );
        // Log refund in transaction history
        $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
            'user_id'    => $order->user_id,
            'amount'     => $order->agent_cost,
            'type'       => 'refund_failed_order',
            'status'     => 'approved',
            'reference'  => 'REFUND-' . $order->ref,
            'created_at' => current_time( 'mysql' ),
        ) );
        $wpdb->update( $wpdb->prefix . 'highest_topup_history', array( 'balance_after' => $wallet + $order->agent_cost ), array( 'id' => $wpdb->insert_id ) );
        // Mark order as REFUNDED
        $wpdb->update(
            $wpdb->prefix . 'highest_orders',
            array( 'status' => 'REFUNDED', 'api_response' => wp_json_encode( array( 'note' => 'Wallet refunded by admin on ' . current_time( 'mysql' ) ) ) ),
            array( 'id' => $order->id )
        );
        wp_redirect( admin_url( 'admin.php?page=highest-orders-payments&refund=success' ) );
        exit;
    }

    public function handle_deduction() {
        if ( ! current_user_can( 'manage_options' ) || ! isset( $_POST['deduct_submit'] ) ) return;
        $agent_id = intval( $_POST['agent_id'] );
        $type = sanitize_text_field( $_POST['deduct_type'] );
        $action_type = sanitize_text_field( $_POST['action_type'] ?? 'deduct' );
        $amount = floatval( $_POST['deduct_amount'] );
        if ( $amount <= 0 ) return;
        if ( $type === 'wallet' ) {
            $wallet = (float) get_user_meta( $agent_id, 'highest_wallet', true );
            $new_wallet = $action_type === 'add' ? $wallet + $amount : max( 0, $wallet - $amount );
            update_user_meta( $agent_id, 'highest_wallet', $new_wallet );
            global $wpdb;
            $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                'user_id'   => $agent_id,
                'amount'    => $amount,
                'type'      => 'admin_' . $action_type . '_wallet',
                'status'    => 'approved',
                'reference' => 'ADMIN-' . time(),
                'created_at'=> current_time( 'mysql' ),
            ) );
            $wpdb->update( $wpdb->prefix . 'highest_topup_history', array( 'balance_after' => $new_wallet ), array( 'id' => $wpdb->insert_id ) );
        } elseif ( $type === 'profit' ) {
            $profit = (float) get_user_meta( $agent_id, 'highest_profit', true );
            $new_profit = $action_type === 'add' ? $profit + $amount : max( 0, $profit - $amount );
            update_user_meta( $agent_id, 'highest_profit', $new_profit );
            $change = $action_type === 'add' ? $amount : -$amount;
            $this->add_profit_history( $agent_id, $change, 'admin_adjust', 'ADMIN-' . time(), "Admin {$action_type} profit: GH₵{$amount}" );
        }
        wp_redirect( admin_url( 'admin.php?page=highest-all-agents&updated=1' ) );
        exit;
    }

    public function handle_manual_topup_approval() {
        if ( ! current_user_can( 'manage_options' ) ) return;
        if ( isset( $_POST['approve_manual_topup'] ) ) {
            $user_id = intval( $_POST['user_id'] );
            $request_id = sanitize_text_field( $_POST['request_id'] );
            $requests = get_user_meta( $user_id, 'highest_manual_topup_requests', true ) ?: array();
            foreach ( $requests as &$req ) {
                if ( isset( $req['id'] ) && $req['id'] === $request_id && $req['status'] === 'pending' ) {
                    $req['status'] = 'approved';
                    $req['approved_at'] = current_time( 'mysql' );
                    $amount = (float) $req['amount'];
                    $wallet = (float) get_user_meta( $user_id, 'highest_wallet', true );
                    $new_wallet = $wallet + $amount;
                    update_user_meta( $user_id, 'highest_wallet', $new_wallet );
                    global $wpdb;
                    $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                        'user_id'   => $user_id,
                        'amount'    => $amount,
                        'type'      => 'manual',
                        'status'    => 'approved',
                        'reference' => 'MANUAL-' . $request_id,
                        'created_at'=> current_time( 'mysql' ),
                    ) );
                    $wpdb->update( $wpdb->prefix . 'highest_topup_history', array( 'balance_after' => $new_wallet ), array( 'id' => $wpdb->insert_id ) );
                    if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) ) {
                        $agent_phone = $this->get_agent_phone( $user_id );
                        if ( ! empty( $agent_phone ) ) {
                            $this->send_templated_sms( $agent_phone, 'manual_topup_agent', array(
                                'amount'  => number_format( $amount, 2 ),
                                'balance' => number_format( $new_wallet, 2 ),
                            ) );
                        }
                    }
                    break;
                }
            }
            update_user_meta( $user_id, 'highest_manual_topup_requests', $requests );
            wp_redirect( admin_url( 'admin.php?page=highest-manual-topups&approved=1' ) );
            exit;
        }
        if ( isset( $_POST['reject_manual_topup'] ) ) {
            $user_id = intval( $_POST['user_id'] );
            $request_id = sanitize_text_field( $_POST['request_id'] );
            $requests = get_user_meta( $user_id, 'highest_manual_topup_requests', true ) ?: array();
            foreach ( $requests as &$req ) {
                if ( isset( $req['id'] ) && $req['id'] === $request_id && $req['status'] === 'pending' ) {
                    $req['status'] = 'rejected';
                    $req['rejected_at'] = current_time( 'mysql' );
                    break;
                }
            }
            update_user_meta( $user_id, 'highest_manual_topup_requests', $requests );
            wp_redirect( admin_url( 'admin.php?page=highest-manual-topups&rejected=1' ) );
            exit;
        }
    }

    public function handle_toggle_sms_disable() {
        if ( ! current_user_can( 'manage_options' ) || ! isset( $_POST['toggle_sms_disable'] ) ) return;
        $agent_id = intval( $_POST['agent_id'] );
        $current = $this->is_agent_sms_disabled( $agent_id );
        update_user_meta( $agent_id, 'highest_sms_disabled', $current ? 0 : 1 );
        wp_redirect( admin_url( 'admin.php?page=highest-all-agents&updated=1' ) );
        exit;
    }

    public function handle_update_tier() {
        if ( ! current_user_can( 'manage_options' ) || ! isset( $_POST['update_tier'] ) ) return;
        $agent_id = intval( $_POST['agent_id'] );
        $tier = sanitize_text_field( $_POST['tier'] ?? 'standard' );
        update_user_meta( $agent_id, 'highest_agent_tier', $tier );
        wp_redirect( admin_url( 'admin.php?page=highest-all-agents&updated=1' ) );
        exit;
    }

    public function all_agents_page() {
        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        global $wpdb;
        $table = $wpdb->prefix . 'highest_orders';
        ?>
        <div class="wrap">
            <h1>👥 All Agents</h1>
            <?php if ( isset( $_GET['updated'] ) ) : ?>
                <div class="notice notice-success is-dismissible"><p>✅ Agent updated successfully.</p></div>
            <?php endif; ?>
            <div style="overflow-x: auto;">
                <table class="wp-list-table widefat fixed striped" style="min-width: 800px;">
                    <thead>
                        <tr><th>Agent Name</th><th>Tier</th><th>Wallet (₵)</th><th>Profit (₵)</th><th>Sales (₵)</th><th>SMS</th><th>Status</th><th>Wallet Actions</th><th>Profit Actions</th></tr>
                    </thead>
                    <tbody>
                    <?php foreach ( $agents as $agent ) :
                        $wallet = (float) get_user_meta( $agent->ID, 'highest_wallet', true );
                        $profit = (float) get_user_meta( $agent->ID, 'highest_profit', true );
                        $sales = $wpdb->get_var( $wpdb->prepare( "SELECT SUM(selling_price) FROM $table WHERE user_id = %d AND status='DELIVERED'", $agent->ID ) ) ?: 0;
                        $sms_disabled = $this->is_agent_sms_disabled( $agent->ID );
                    ?>
                        <tr>
                            <td><strong><?php echo esc_html( $agent->display_name ); ?></strong><br><small>ID: <?php echo $agent->ID; ?></small></td>
                            <td>
                                <form method="post" style="display:inline;">
                                    <input type="hidden" name="agent_id" value="<?php echo esc_attr( $agent->ID ); ?>">
                                    <input type="hidden" name="update_tier" value="1">
                                    <select name="tier" onchange="this.form.submit()">
                                        <?php $current_tier = get_user_meta( $agent->ID, 'highest_agent_tier', true ) ?: 'standard'; ?>
                                        <?php foreach ( get_option( 'highest_agent_tiers', array( 'standard' => 'Standard' ) ) as $slug => $tname ) : ?>
                                            <option value="<?php echo esc_attr( $slug ); ?>" <?php selected( $current_tier, $slug ); ?>><?php echo esc_html( $tname ); ?></option>
                                        <?php endforeach; ?>
                                    </select>
                                </form>
                            </td>
                            <td>GH₵<?php echo number_format( $wallet, 2 ); ?></td>
                            <td>GH₵<?php echo number_format( $profit, 2 ); ?></td>
                            <td>GH₵<?php echo number_format( $sales, 2 ); ?></td>
                            <td>
                                <form method="post">
                                    <input type="hidden" name="agent_id" value="<?php echo esc_attr( $agent->ID ); ?>">
                                    <input type="hidden" name="toggle_sms_disable" value="1">
                                    <button class="button button-small <?php echo $sms_disabled ? 'button-secondary' : 'button-primary'; ?>">
                                        <?php echo $sms_disabled ? 'Enable SMS' : 'Disable SMS'; ?>
                                    </button>
                                </form>
                             </div>
                            <?php $is_suspended = get_user_meta( $agent->ID, 'highest_agent_suspended', true ) === '1'; ?>
                            <td>
                                <form method="post">
                                    <input type="hidden" name="agent_id" value="<?php echo esc_attr( $agent->ID ); ?>">
                                    <input type="hidden" name="toggle_suspend" value="1">
                                    <button class="button button-small" style="<?php echo $is_suspended ? 'background:#dc2626;border-color:#dc2626;color:#fff;' : ''; ?>" onclick="return confirm('<?php echo $is_suspended ? 'Unsuspend' : 'Suspend'; ?> this agent?');">
                                        <?php echo $is_suspended ? 'Unsuspend' : 'Suspend'; ?>
                                    </button>
                                </form>
                            </td>
                            <td>
                                <form method="post" style="display:inline-block; margin-right:5px;">
                                    <input type="hidden" name="agent_id" value="<?php echo $agent->ID; ?>">
                                    <input type="hidden" name="deduct_type" value="wallet">
                                    <input type="hidden" name="action_type" value="add">
                                    <input type="number" name="deduct_amount" step="0.01" placeholder="Add" style="width:70px;">
                                    <button type="submit" name="deduct_submit" class="button button-small button-primary">+</button>
                                </form>
                                <form method="post" style="display:inline-block;">
                                    <input type="hidden" name="agent_id" value="<?php echo $agent->ID; ?>">
                                    <input type="hidden" name="deduct_type" value="wallet">
                                    <input type="hidden" name="action_type" value="deduct">
                                    <input type="number" name="deduct_amount" step="0.01" placeholder="Deduct" style="width:70px;">
                                    <button type="submit" name="deduct_submit" class="button button-small">-</button>
                                </form>
                             </div>
                            <td>
                                <form method="post" style="display:inline-block; margin-right:5px;">
                                    <input type="hidden" name="agent_id" value="<?php echo $agent->ID; ?>">
                                    <input type="hidden" name="deduct_type" value="profit">
                                    <input type="hidden" name="action_type" value="add">
                                    <input type="number" name="deduct_amount" step="0.01" placeholder="Add" style="width:70px;">
                                    <button type="submit" name="deduct_submit" class="button button-small button-primary">+</button>
                                </form>
                                <form method="post" style="display:inline-block;">
                                    <input type="hidden" name="agent_id" value="<?php echo $agent->ID; ?>">
                                    <input type="hidden" name="deduct_type" value="profit">
                                    <input type="hidden" name="action_type" value="deduct">
                                    <input type="number" name="deduct_amount" step="0.01" placeholder="Deduct" style="width:70px;">
                                    <button type="submit" name="deduct_submit" class="button button-small">-</button>
                                </form>
                             </div>
                         </tr>
                    <?php endforeach; ?>
                    </tbody>
                </table>
            </div>
            <style>
                .wp-list-table td, .wp-list-table th { vertical-align: middle; }
                .wp-list-table input[type="number"] { width: 70px; }
                .wp-list-table .button-small { padding: 0 8px; line-height: 28px; height: 28px; }
            </style>
        </div>
        <?php
    }

    public function manual_topups_page() {
        $message = '';
        if ( isset( $_GET['approved'] ) ) $message = '<div class="notice notice-success"><p>✅ Manual top-up approved and wallet credited.</p></div>';
        if ( isset( $_GET['rejected'] ) ) $message = '<div class="notice notice-info"><p>❌ Manual top-up rejected.</p></div>';
        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        ?>
        <div class="wrap">
            <h1>📲 Manual Top-up Requests</h1>
            <p>Review, approve or reject agent manual top-up requests. Approve credits the wallet immediately.</p>
            <?php echo $message; ?>
            <table class="wp-list-table widefat fixed striped">
                <thead><tr><th>Agent</th><th>Date &amp; Time</th><th>Amount (GH₵)</th><th>Status</th><th>Actions</th></tr></thead>
                <tbody>
                <?php
                $has_any = false;
                foreach ( $agents as $agent ) {
                    $requests = get_user_meta( $agent->ID, 'highest_manual_topup_requests', true ) ?: array();
                    foreach ( $requests as $req ) {
                        if ( ! is_array( $req ) ) continue;
                        $has_any = true;
                        $status = $req['status'] ?? 'pending';
                        ?>
                        <tr>
                            <td><strong><?php echo esc_html( $agent->display_name ); ?></strong><br><small>ID: <?php echo $agent->ID; ?></small></td>
                            <td><?php echo esc_html( $req['date'] ?? '—' ); ?></td>
                            <td><strong><?php echo number_format( (float) ( $req['amount'] ?? 0 ), 2 ); ?></strong></td>
                            <td><span style="padding:4px 12px;border-radius:9999px;font-size:13px;<?php echo $status === 'approved' ? 'background:#10b981;color:white;' : ( $status === 'rejected' ? 'background:#ef4444;color:white;' : 'background:#eab308;color:#1f2937;' ); ?>"><?php echo esc_html( ucfirst( $status ) ); ?></span></td>
                            <td><?php if ( $status === 'pending' ) : ?>
                                <form method="post" style="display:inline-block;margin-right:4px;"><input type="hidden" name="user_id" value="<?php echo esc_attr( $agent->ID ); ?>"><input type="hidden" name="request_id" value="<?php echo esc_attr( $req['id'] ?? '' ); ?>"><button type="submit" name="approve_manual_topup" class="button button-primary">Approve &amp; Credit</button></form>
                                <form method="post" style="display:inline-block;"><input type="hidden" name="user_id" value="<?php echo esc_attr( $agent->ID ); ?>"><input type="hidden" name="request_id" value="<?php echo esc_attr( $req['id'] ?? '' ); ?>"><button type="submit" name="reject_manual_topup" onclick="return confirm('Reject this request?');" class="button button-secondary">Reject</button></form>
                            <?php else : echo 'Processed'; endif; ?> </div>
                         </tr>
                        <?php
                    }
                }
                if ( ! $has_any ) echo '<tr><td colspan="5" style="text-align:center;padding:40px;">No manual top-up requests yet.</td></tr>';
                ?>
                </tbody>
            </table>
        </div>
        <?php
    }

    public function topup_history_page() {
        global $wpdb;
        $table_topup = $wpdb->prefix . 'highest_topup_history';

        $filter_type  = isset( $_GET['type'] )     ? sanitize_text_field( $_GET['type'] )     : '';
        $filter_agent = isset( $_GET['agent_id'] ) ? intval( $_GET['agent_id'] )               : 0;
        $date_from    = isset( $_GET['date_from'] )? sanitize_text_field( $_GET['date_from'] ) : '';
        $date_to      = isset( $_GET['date_to'] )  ? sanitize_text_field( $_GET['date_to'] )   : '';

        $where = "WHERE 1=1";
        if ( $filter_type )  $where .= $wpdb->prepare( " AND type = %s", $filter_type );
        if ( $filter_agent ) $where .= $wpdb->prepare( " AND user_id = %d", $filter_agent );
        if ( $date_from )    $where .= $wpdb->prepare( " AND DATE(created_at) >= %s", $date_from );
        if ( $date_to )      $where .= $wpdb->prepare( " AND DATE(created_at) <= %s", $date_to );

        $records = $wpdb->get_results( "SELECT * FROM $table_topup $where ORDER BY created_at DESC LIMIT 500" );

        $total_in = 0; $total_out = 0;
        foreach ( $records as $r ) {
            if ( $r->amount >= 0 ) $total_in  += (float) $r->amount;
            else                   $total_out += abs( (float) $r->amount );
        }

        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        ?>
        <div class="wrap">
            <h1>💳 Wallet Transaction History</h1>
            <form method="get" style="margin-bottom:20px;display:flex;flex-wrap:wrap;gap:8px;align-items:center;">
                <input type="hidden" name="page" value="highest-topup-history">
                <select name="type">
                    <option value="">All Types</option>
                    <option value="paystack"           <?php selected( $filter_type, 'paystack' ); ?>>Top-up (Paystack)</option>
                    <option value="manual"             <?php selected( $filter_type, 'manual' ); ?>>Top-up (Manual)</option>
                    <option value="admin_add_wallet"   <?php selected( $filter_type, 'admin_add_wallet' ); ?>>Admin Add</option>
                    <option value="purchase"           <?php selected( $filter_type, 'purchase' ); ?>>Purchase (deduction)</option>
                    <option value="refund_failed_order"<?php selected( $filter_type, 'refund_failed_order' ); ?>>Refund</option>
                    <option value="daily_login_bonus"  <?php selected( $filter_type, 'daily_login_bonus' ); ?>>Daily Bonus</option>
                </select>
                <select name="agent_id">
                    <option value="">All Agents</option>
                    <?php foreach ( $agents as $a ) : ?>
                        <option value="<?php echo $a->ID; ?>" <?php selected( $filter_agent, $a->ID ); ?>><?php echo esc_html( $a->display_name ); ?></option>
                    <?php endforeach; ?>
                </select>
                <input type="date" name="date_from" value="<?php echo esc_attr( $date_from ); ?>">
                <input type="date" name="date_to"   value="<?php echo esc_attr( $date_to ); ?>">
                <button type="submit" class="button">Filter</button>
                <a href="?page=highest-topup-history" class="button">Reset</a>
            </form>

            <div style="display:flex;gap:20px;margin-bottom:15px;flex-wrap:wrap;">
                <div style="background:#fff;padding:15px 20px;border-radius:8px;box-shadow:0 1px 3px rgba(0,0,0,.1);">
                    <strong>Total Deposits:</strong> <span style="color:#10b981;">GH₵<?php echo number_format( $total_in, 2 ); ?></span>
                </div>
                <div style="background:#fff;padding:15px 20px;border-radius:8px;box-shadow:0 1px 3px rgba(0,0,0,.1);">
                    <strong>Total Deductions:</strong> <span style="color:#ef4444;">GH₵<?php echo number_format( $total_out, 2 ); ?></span>
                </div>
                <div style="background:#fff;padding:15px 20px;border-radius:8px;box-shadow:0 1px 3px rgba(0,0,0,.1);">
                    <strong>Net:</strong> <span style="color:#6366f1;">GH₵<?php echo number_format( $total_in - $total_out, 2 ); ?></span>
                </div>
            </div>

            <table class="wp-list-table widefat fixed striped">
                <thead><tr>
                    <th>Date</th><th>Agent</th><th>Amount (GH₵)</th><th>Type</th><th>Reference</th><th>Balance After</th><th>Current Wallet</th>
                </tr></thead>
                <tbody>
                <?php if ( empty( $records ) ) : ?>
                    <tr><td colspan="6" style="text-align:center;padding:20px;">No transactions found.</td></tr>
                <?php else : foreach ( $records as $rec ) :
                    $agent  = get_user_by( 'id', $rec->user_id );
                    $color  = $rec->amount >= 0 ? '#10b981' : '#ef4444';
                    $sign   = $rec->amount >= 0 ? '+' : '';
                    $tlabel = ucwords( str_replace( '_', ' ', $rec->type ) );
                    $cur_wallet = (float) get_user_meta( $rec->user_id, 'highest_wallet', true );
                ?>
                    <tr>
                        <td><?php echo esc_html( $rec->created_at ); ?></td>
                        <td><strong><?php echo $agent ? esc_html( $agent->display_name ) : '—'; ?></strong></td>
                        <td style="color:<?php echo $color; ?>;font-weight:bold;"><?php echo $sign . number_format( (float)$rec->amount, 2 ); ?></td>
                        <td><span style="background:#e0f2fe;color:#0369a1;padding:3px 8px;border-radius:4px;font-weight:bold;font-size:12px;"><?php echo esc_html( strtoupper( $rec->type ) ); ?></span></td>
                        <td><code><?php echo esc_html( $rec->reference ); ?></code></td>
                        <td><strong>GH₵<?php echo number_format( (float)$rec->balance_after, 2 ); ?></strong></td>
                        <td><strong>GH₵<?php echo number_format( $cur_wallet, 2 ); ?></strong></td>
                    </tr>
                <?php endforeach; endif; ?>
                </tbody>
            </table>
        </div>
        <?php
    }

    public function announcements_page() {
        if ( isset( $_POST['highest_add_announcement'] ) ) {
            $announcements = get_option( 'highest_announcements', array() );
            $announcements[] = array(
                'id'      => uniqid(),
                'title'   => sanitize_text_field( $_POST['title'] ),
                'message' => sanitize_textarea_field( $_POST['message'] ),
                'date'    => current_time( 'mysql' ),
            );
            update_option( 'highest_announcements', $announcements );
            echo '<div class="notice notice-success is-dismissible"><p>✅ Announcement published.</p></div>';
        }
        if ( isset( $_POST['highest_delete_announcement'] ) && isset( $_POST['announcement_id'] ) && current_user_can( 'manage_options' ) ) {
            $delete_id = sanitize_text_field( $_POST['announcement_id'] );
            $announcements = get_option( 'highest_announcements', array() );
            $announcements = array_values( array_filter( $announcements, function( $a ) use ( $delete_id ) {
                return $a['id'] !== $delete_id;
            } ) );
            update_option( 'highest_announcements', $announcements );
            echo '<div class="notice notice-success is-dismissible"><p>✅ Announcement deleted.</p></div>';
        }
        $announcements = get_option( 'highest_announcements', array() );
        ?>
        <div class="wrap">
            <h1>📢 Announcements for Agents</h1>
            <form method="post">
                <p><strong>Title</strong></p>
                <input type="text" name="title" required style="width:100%;max-width:600px;">
                <p><strong>Message</strong></p>
                <textarea name="message" rows="5" required style="width:100%;max-width:600px;"></textarea><br><br>
                <button type="submit" name="highest_add_announcement" class="button button-primary">Publish Announcement</button>
            </form>
            <hr>
            <h2>Existing Announcements</h2>
            <?php if ( $announcements ) : ?>
                <ul style="max-width:800px;">
                <?php foreach ( array_reverse( $announcements ) as $a ) : ?>
                    <li style="background:#fff;padding:15px;margin-bottom:10px;border:1px solid #ccd0d4;border-radius:4px;display:flex;justify-content:space-between;align-items:start;">
                        <div>
                            <strong><?php echo esc_html( $a['title'] ); ?></strong> — <?php echo esc_html( $a['message'] ); ?><br>
                            <small style="color:#666;">(<?php echo $a['date']; ?>)</small>
                        </div>
                        <form method="post" style="margin:0;">
                            <input type="hidden" name="announcement_id" value="<?php echo esc_attr( $a['id'] ); ?>">
                            <button type="submit" name="highest_delete_announcement" class="button button-link-delete" style="color:#b32d2e;" onclick="return confirm('Delete this announcement?');">Delete</button>
                        </form>
                    </li>
                <?php endforeach; ?>
                </ul>
            <?php else : ?>
                <p>No announcements yet.</p>
            <?php endif; ?>
        </div>
        <?php
    }

    public function bulk_sms_page() {
        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        $sms_enabled = $this->is_sms_enabled();
        ?>
        <div class="wrap">
            <h1>📱 Bulk SMS to Agents</h1>
            <?php if ( ! $sms_enabled ) : ?>
                <div class="notice notice-warning"><p>⚠️ SMS notifications are currently <strong>disabled</strong> in <a href="?page=highest-settings">plugin settings</a>. Enable them to send bulk messages.</p></div>
            <?php endif; ?>
            <p>Send a message to all agents or selected agents using Hubtel SMS (legacy endpoint). Agents with SMS disabled will be automatically skipped.</p>
            <div id="bulk-sms-status"></div>
            <form id="bulk-sms-form" method="post">
                <?php wp_nonce_field( 'highest_bulk_sms', 'bulk_sms_nonce' ); ?>
                <table class="form-table">
                    <tr><th scope="row">Target Audience</th><td><select id="sms-target" name="target"><option value="all_agents">All Agents (skip disabled SMS)</option><option value="selected_agents">Selected Agents Only</option></select></td></tr>
                    <tr><th scope="row">Select Agents</th><td><div id="agent-list" style="display:none; max-height:200px; overflow-y:auto; border:1px solid #ddd; padding:8px;"><?php foreach ( $agents as $agent ) : $phone = get_user_meta( $agent->ID, 'phone', true ); if ( empty( $phone ) ) continue; $disabled = $this->is_agent_sms_disabled( $agent->ID ); ?><label style="display:block; margin-bottom:5px;"><input type="checkbox" name="agent_ids[]" value="<?php echo esc_attr( $agent->ID ); ?>"> <?php echo esc_html( $agent->display_name ); ?> (<?php echo esc_html( $phone ); ?>)<?php if ( $disabled ) echo ' <span style="color:red;"> (SMS disabled)</span>'; ?></label><?php endforeach; ?></div></td></tr>
                    <tr><th scope="row">Message</th><td><textarea id="sms-message" rows="5" cols="50" style="width:100%; max-width:500px;" placeholder="Type your message here..."></textarea><p class="description">SMS length limit: 160 characters per message (longer messages may be split).</p></td></tr>
                </table>
                <p class="submit"><button type="button" id="send-bulk-sms" class="button button-primary" <?php echo $sms_enabled ? '' : 'disabled'; ?>>Send SMS</button></p>
            </form>
        </div>
        <script>
        jQuery(document).ready(function($) {
            $('#sms-target').on('change', function() { $('#agent-list').toggle(this.value === 'selected_agents'); });
            $('#send-bulk-sms').on('click', function() {
                var target = $('#sms-target').val(), message = $('#sms-message').val().trim();
                if (!message) { alert('Please enter a message.'); return; }
                var agentIds = [];
                if (target === 'selected_agents') {
                    var checkboxes = $('input[name="agent_ids[]"]:checked');
                    if (checkboxes.length === 0) { alert('Please select at least one agent.'); return; }
                    checkboxes.each(function() { agentIds.push($(this).val()); });
                }
                var btn = $(this); btn.prop('disabled', true).text('Sending...');
                var fd = new FormData();
                fd.append('action', 'highest_bulk_sms');
                fd.append('nonce', '<?php echo wp_create_nonce( "highest_data_plug" ); ?>');
                fd.append('target', target); fd.append('message', message);
                if (agentIds.length) agentIds.forEach(function(id) { fd.append('agent_ids[]', id); });
                fetch(ajaxurl, { method:'POST', credentials:'same-origin', body:fd })
                .then(r => r.json()).then(res => {
                    if (res.success) { $('#bulk-sms-status').html('<div class="notice notice-success"><p>✅ Sent: ' + res.data.success_count + ' | Failed: ' + res.data.fail_count + '</p></div>'); if (res.data.errors.length) $('#bulk-sms-status').append('<div class="notice notice-warning"><p>Errors:<br>' + res.data.errors.join('<br>') + '</p></div>'); }
                    else { $('#bulk-sms-status').html('<div class="notice notice-error"><p>❌ ' + (res.data || 'Unknown error') + '</p></div>'); }
                    btn.prop('disabled', false).text('Send SMS');
                }).catch(() => { $('#bulk-sms-status').html('<div class="notice notice-error"><p>Network error. Please try again.</p></div>'); btn.prop('disabled', false).text('Send SMS'); });
            });
        });
        </script>
        <?php
    }

    public function ajax_bulk_sms() {
        $this->verify_ajax_request();
        if ( ! current_user_can( 'manage_options' ) ) wp_send_json_error( 'Unauthorized.' );
        if ( ! $this->is_sms_enabled() ) wp_send_json_error( 'SMS notifications are globally disabled.' );

        $message = sanitize_textarea_field( $_POST['message'] ?? '' );
        $target = sanitize_text_field( $_POST['target'] ?? 'all_agents' );
        if ( empty( $message ) ) wp_send_json_error( 'Message cannot be empty.' );

        $recipients = array();
        if ( $target === 'all_agents' ) {
            $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
            foreach ( $agents as $agent ) {
                if ( $this->is_agent_sms_disabled( $agent->ID ) ) continue;
                $phone = $this->get_agent_phone( $agent->ID );
                if ( ! empty( $phone ) ) $recipients[] = $phone;
            }
        } elseif ( $target === 'selected_agents' && isset( $_POST['agent_ids'] ) ) {
            $agent_ids = array_map( 'intval', $_POST['agent_ids'] );
            foreach ( $agent_ids as $aid ) {
                if ( $this->is_agent_sms_disabled( $aid ) ) continue;
                $phone = $this->get_agent_phone( $aid );
                if ( ! empty( $phone ) ) $recipients[] = $phone;
            }
        }
        if ( empty( $recipients ) ) wp_send_json_error( 'No valid recipient phone numbers found (or all selected agents have SMS disabled).' );

        $success_count = 0; $fail_count = 0; $errors = array();
        foreach ( $recipients as $phone ) {
            $result = $this->send_templated_sms( $phone, 'bulk_agents', array( 'message' => $message ) );
            if ( $result === true ) {
                $success_count++;
            } else {
                $fail_count++;
                $errors[] = "$phone: $result";
            }
            usleep( 200000 );
        }
        wp_send_json_success( array( 'success_count' => $success_count, 'fail_count' => $fail_count, 'errors' => $errors ) );
    }

    public function bulk_sms_customers_page() {
        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        $sms_enabled = $this->is_sms_enabled();
        ?>
        <div class="wrap">
            <h1>📱 Bulk SMS to Customers</h1>
            <?php if ( ! $sms_enabled ) : ?><div class="notice notice-warning"><p>⚠️ SMS notifications are currently <strong>disabled</strong> in <a href="?page=highest-settings">plugin settings</a>. Enable them to send bulk messages.</p></div><?php endif; ?>
            <p>Send SMS to customers based on their order history. For customers of an agent, the agent's store link will be automatically appended. For walk‑in customers, the main site URL is appended.</p>
            <div id="bulk-sms-customers-status"></div>
            <form id="bulk-sms-customers-form">
                <table class="form-table">
                    <tr><th scope="row">Target Audience</th><td><select id="customer-target" name="target"><option value="agent_customers">Customers of a specific agent</option><option value="walkin_customers">Walk‑in customers (no agent)</option><option value="all_customers">All customers (both agent and walk‑in)</option><option value="custom">Custom phone numbers (enter below)</option></select></td></tr>
                    <tr id="agent-select-row" style="display:none;"><th scope="row">Select Agent</th><td><select id="customer-agent-id" name="agent_id"><option value="">-- Select Agent --</option><?php foreach ( $agents as $agent ) : ?><option value="<?php echo esc_attr( $agent->ID ); ?>"><?php echo esc_html( $agent->display_name ); ?></option><?php endforeach; ?></select></td></tr>
                    <tr id="custom-numbers-row" style="display:none;"><th scope="row">Phone Numbers (one per line)</th><td><textarea id="custom-numbers" rows="6" cols="50" style="width:100%; max-width:500px;" placeholder="0551234567&#10;0249876543&#10;..."></textarea><p class="description">Enter Ghana phone numbers (10 digits) each on a new line.</p></td></tr>
                    <tr><th scope="row">Message</th><td><textarea id="customer-sms-message" rows="5" cols="50" style="width:100%; max-width:500px;" placeholder="Type your message here..."></textarea><p class="description"><strong>Note:</strong> The appropriate link (agent store or main site) will be added automatically. Do not add "Track at:" yourself unless you want duplication.</p></td></tr>
                </table>
                <p class="submit"><button type="button" id="send-bulk-sms-customers" class="button button-primary" <?php echo $sms_enabled ? '' : 'disabled'; ?>>Send SMS to Customers</button></p>
            </form>
        </div>
        <script>
        jQuery(document).ready(function($) {
            $('#customer-target').on('change', function() {
                var val = $(this).val();
                $('#agent-select-row').toggle(val === 'agent_customers');
                $('#custom-numbers-row').toggle(val === 'custom');
            }).trigger('change');
            $('#send-bulk-sms-customers').on('click', function() {
                var target = $('#customer-target').val(), message = $('#customer-sms-message').val().trim();
                if (!message) { alert('Please enter a message.'); return; }
                var agentId = 0, customNumbers = '';
                if (target === 'agent_customers') { agentId = parseInt($('#customer-agent-id').val()); if (!agentId) { alert('Please select an agent.'); return; } }
                if (target === 'custom') { customNumbers = $('#custom-numbers').val(); if (!customNumbers.trim()) { alert('Please enter at least one phone number.'); return; } }
                var btn = $(this); btn.prop('disabled', true).text('Sending...');
                var fd = new FormData();
                fd.append('action', 'highest_bulk_sms_customers');
                fd.append('nonce', '<?php echo wp_create_nonce( "highest_data_plug" ); ?>');
                fd.append('target', target); fd.append('message', message);
                if (agentId) fd.append('agent_id', agentId);
                if (customNumbers) fd.append('custom_numbers', customNumbers);
                fetch(ajaxurl, { method:'POST', credentials:'same-origin', body:fd })
                .then(r => r.json()).then(res => {
                    if (res.success) { $('#bulk-sms-customers-status').html('<div class="notice notice-success"><p>✅ Sent: ' + res.data.success_count + ' | Failed: ' + res.data.fail_count + '</p></div>'); if (res.data.errors.length) $('#bulk-sms-customers-status').append('<div class="notice notice-warning"><p>Errors:<br>' + res.data.errors.join('<br>') + '</p></div>'); }
                    else { $('#bulk-sms-customers-status').html('<div class="notice notice-error"><p>❌ ' + (res.data || 'Unknown error') + '</p></div>'); }
                    btn.prop('disabled', false).text('Send SMS to Customers');
                }).catch(() => { $('#bulk-sms-customers-status').html('<div class="notice notice-error"><p>Network error. Please try again.</p></div>'); btn.prop('disabled', false).text('Send SMS to Customers'); });
            });
        });
        </script>
        <?php
    }

    public function ajax_bulk_sms_customers() {
        $this->verify_ajax_request();
        if ( ! current_user_can( 'manage_options' ) ) wp_send_json_error( 'Unauthorized.' );
        if ( ! $this->is_sms_enabled() ) wp_send_json_error( 'SMS notifications are globally disabled.' );

        $target      = sanitize_text_field( $_POST['target'] ?? '' );
        $message_base = sanitize_textarea_field( $_POST['message'] ?? '' );
        $custom_numbers = sanitize_textarea_field( $_POST['custom_numbers'] ?? '' );
        $agent_id    = intval( $_POST['agent_id'] ?? 0 );
        if ( empty( $message_base ) ) wp_send_json_error( 'Message cannot be empty.' );

        $phone_numbers = array();
        if ( $target === 'custom' ) {
            $lines = explode( "\n", $custom_numbers );
            foreach ( $lines as $line ) {
                $phone = trim( preg_replace( '/\D/', '', $line ) );
                if ( ! empty( $phone ) ) $phone_numbers[] = $phone;
            }
        } elseif ( $target === 'agent_customers' && $agent_id > 0 ) {
            global $wpdb;
            $table_orders = $wpdb->prefix . 'highest_orders';
            $phone_numbers = $wpdb->get_col( $wpdb->prepare( "SELECT DISTINCT phone FROM $table_orders WHERE user_id = %d AND order_type != 'walkin' AND phone != ''", $agent_id ) );
        } elseif ( $target === 'walkin_customers' ) {
            global $wpdb;
            $table_orders = $wpdb->prefix . 'highest_orders';
            $phone_numbers = $wpdb->get_col( "SELECT DISTINCT phone FROM $table_orders WHERE order_type = 'walkin' AND phone != ''" );
        } elseif ( $target === 'all_customers' ) {
            global $wpdb;
            $table_orders = $wpdb->prefix . 'highest_orders';
            $phone_numbers = $wpdb->get_col( "SELECT DISTINCT phone FROM $table_orders WHERE phone != ''" );
        } else {
            wp_send_json_error( 'Invalid target or missing agent selection.' );
        }
        if ( empty( $phone_numbers ) ) wp_send_json_error( 'No phone numbers found for the selected target.' );

        $link_url = '';
        if ( $target === 'agent_customers' && $agent_id > 0 ) {
            $store_name = get_user_meta( $agent_id, 'highest_store_name', true );
            $link_url = $store_name ? home_url( '/store/' . sanitize_title( $store_name ) ) : home_url( '/' );
        } elseif ( $target === 'walkin_customers' || $target === 'all_customers' ) {
            $link_url = home_url( '/' );
        }
        $final_message = $message_base;
        if ( ! empty( $link_url ) ) {
            if ( stripos( $final_message, 'track at:' ) === false ) $final_message .= " Track at: {$link_url}";
            else $final_message .= " {$link_url}";
        }

        $success_count = 0; $fail_count = 0; $errors = array();
        foreach ( $phone_numbers as $phone ) {
            $result = $this->send_templated_sms( $phone, 'bulk_customers', array( 'message' => $final_message ) );
            if ( $result === true ) $success_count++; else { $fail_count++; $errors[] = "$phone: $result"; }
            usleep( 200000 );
        }
        wp_send_json_success( array( 'success_count' => $success_count, 'fail_count' => $fail_count, 'errors' => $errors ) );
    }

    public function profit_history_page() {
        global $wpdb;
        $table_profit = $wpdb->prefix . 'highest_profit_history';
        $filter_agent = isset( $_GET['agent_id'] ) ? intval( $_GET['agent_id'] ) : 0;
        $date_from = isset( $_GET['date_from'] ) ? sanitize_text_field( $_GET['date_from'] ) : '';
        $date_to = isset( $_GET['date_to'] ) ? sanitize_text_field( $_GET['date_to'] ) : '';

        $where = "WHERE 1=1";
        if ( $filter_agent ) $where .= $wpdb->prepare( " AND user_id = %d", $filter_agent );
        if ( $date_from ) $where .= $wpdb->prepare( " AND DATE(created_at) >= %s", $date_from );
        if ( $date_to ) $where .= $wpdb->prepare( " AND DATE(created_at) <= %s", $date_to );

        // Handle CSV export
        if ( isset( $_GET['export_csv'] ) && current_user_can( 'manage_options' ) ) {
            $export_records = $wpdb->get_results( "SELECT * FROM $table_profit $where ORDER BY created_at DESC" );

            header( 'Content-Type: text/csv; charset=UTF-8' );
            header( 'Content-Disposition: attachment; filename=agent-profit-history-' . date( 'Y-m-d' ) . '.csv' );

            $output = fopen( 'php://output', 'w' );
            fputcsv( $output, array( 'Date', 'Agent', 'Amount (GH₵)', 'Type', 'Reference', 'Description' ) );

            $total_earned    = 0;
            $total_withdrawn = 0;

            foreach ( $export_records as $rec ) {
                $agent      = get_user_by( 'id', $rec->user_id );
                $agent_name = $agent ? $agent->display_name : 'User ' . $rec->user_id;
                fputcsv( $output, array(
                    $rec->created_at,
                    $agent_name,
                    number_format( $rec->amount, 2 ),
                    $rec->type,
                    $rec->reference,
                    $rec->description,
                ) );
                if ( $rec->amount > 0 ) {
                    $total_earned += $rec->amount;
                } else {
                    $total_withdrawn += abs( $rec->amount );
                }
            }

            // Summary rows
            fputcsv( $output, array( '' ) );
            fputcsv( $output, array( 'TOTAL EARNED',    '', number_format( $total_earned, 2 ) ) );
            fputcsv( $output, array( 'TOTAL WITHDRAWN', '', '-' . number_format( $total_withdrawn, 2 ) ) );
            fputcsv( $output, array( 'NET PROFIT',      '', number_format( $total_earned - $total_withdrawn, 2 ) ) );

            fclose( $output );
            exit;
        }

        $records = $wpdb->get_results( "SELECT * FROM $table_profit $where ORDER BY created_at DESC LIMIT 500" );
        $agents = get_users( array( 'meta_key' => 'is_highest_agent', 'meta_value' => 'yes' ) );
        ?>
        <div class="wrap">
            <h1>💰 Agent Profit History</h1>
            <form method="get" style="margin-bottom:20px;">
                <input type="hidden" name="page" value="highest-profit-history">
                <select name="agent_id"><option value="">All Agents</option><?php foreach ( $agents as $agent ) : ?><option value="<?php echo esc_attr( $agent->ID ); ?>" <?php selected( $filter_agent, $agent->ID ); ?>><?php echo esc_html( $agent->display_name ); ?></option><?php endforeach; ?></select>
                <input type="date" name="date_from" value="<?php echo esc_attr( $date_from ); ?>" placeholder="From">
                <input type="date" name="date_to" value="<?php echo esc_attr( $date_to ); ?>" placeholder="To">
                <button type="submit" class="button">Filter</button>
                <a href="<?php echo admin_url( 'admin.php?page=highest-profit-history' ); ?>" class="button">Reset</a>
                <a href="<?php echo add_query_arg( 'export_csv', '1' ); ?>" class="button">📥 Download CSV</a>
            </form>
            <table class="wp-list-table widefat fixed striped">
                <thead><tr><th>Date</th><th>Agent</th><th>Amount (GH₵)</th><th>Type</th><th>Reference</th><th>Description</th></tr></thead>
                <tbody>
                <?php if ( empty( $records ) ) : ?><tr><td colspan="6" style="text-align:center;">No profit history found.<?php echo $filter_agent ? ' for this agent' : ''; ?></td></tr><?php else : foreach ( $records as $rec ) : $agent = get_user_by( 'id', $rec->user_id ); $amount_class = $rec->amount >= 0 ? 'color:green;' : 'color:red;'; ?>
                    <tr>
                        <td><?php echo esc_html( $rec->created_at ); ?></td>
                        <td><?php echo $agent ? esc_html( $agent->display_name ) : 'User ' . $rec->user_id; ?></td>
                        <td><strong style="<?php echo $amount_class; ?>"><?php echo ( $rec->amount >= 0 ? '+' : '' ) . number_format( $rec->amount, 2 ); ?></strong></td>
                        <td><?php echo esc_html( strtoupper( $rec->type ) ); ?></td>
                        <td><code><?php echo esc_html( $rec->reference ); ?></code></td>
                        <td><?php echo esc_html( $rec->description ); ?></td>
                    </tr>
                <?php endforeach; endif; ?>
                </tbody>
            </table>
        </div>
        <?php
    }

    public function ajax_hubtel_get_sender_ids() {
        if ( ! wp_verify_nonce( $_POST['nonce'] ?? '', 'highest_data_plug' ) ) wp_send_json_error( 'Security fail' );
        if ( ! current_user_can( 'manage_options' ) ) wp_send_json_error( 'Unauthorized' );
        $client_id = get_option( 'highest_hubtel_client_id', '' );
        $client_secret = get_option( 'highest_hubtel_client_secret', '' );
        if ( empty( $client_id ) || empty( $client_secret ) ) wp_send_json_error( 'Hubtel credentials missing' );
        $url = 'https://smsc.hubtel.com/v1/senderids?clientid=' . urlencode( $client_id ) . '&clientsecret=' . urlencode( $client_secret );
        $response = wp_remote_get( $url, array( 'timeout' => 15 ) );
        if ( is_wp_error( $response ) ) wp_send_json_error( $response->get_error_message() );
        $body = json_decode( wp_remote_retrieve_body( $response ), true );
        wp_send_json_success( $body );
    }

    public function ajax_register_agent() {
        if ( ! check_ajax_referer( 'highest_register', 'nonce', false ) ) wp_send_json_error( 'Security check failed.' );

        // Registration control checks (run before creating any user)
        if ( get_option( 'highest_pause_agent_registration', 'no' ) === 'yes' ) {
            wp_send_json_error( 'Agent registration is currently paused.' );
        }
        $referrer_id = intval( $_POST['ref'] ?? 0 );
        if ( get_option( 'highest_require_referral_to_register', 'no' ) === 'yes' && $referrer_id <= 0 ) {
            wp_send_json_error( 'Registration is only allowed via a referral link.' );
        }
        if ( $referrer_id > 0 ) {
            $referrer = get_user_by( 'id', $referrer_id );
            if ( ! $referrer || get_user_meta( $referrer->ID, 'is_highest_agent', true ) !== 'yes' ) {
                wp_send_json_error( 'Invalid referral link.' );
            }
        }

        $name = sanitize_text_field( $_POST['name'] ?? '' );
        $phone = sanitize_text_field( $_POST['phone'] ?? '' );
        $email = sanitize_email( $_POST['email'] ?? '' );
        $password = $_POST['password'] ?? '';
        if ( empty( $email ) || ! is_email( $email ) ) wp_send_json_error( 'Valid email required.' );
        if ( email_exists( $email ) ) wp_send_json_error( 'Email already exists.' );
        if ( empty( $password ) || strlen( $password ) < 8 ) wp_send_json_error( 'Password must be at least 8 characters.' );
        $user_id = wp_create_user( $email, $password, $email );
        if ( is_wp_error( $user_id ) ) wp_send_json_error( $user_id->get_error_message() );
        wp_update_user( array( 'ID' => $user_id, 'first_name' => $name, 'display_name' => $name ) );
        update_user_meta( $user_id, 'phone', $phone );
        update_user_meta( $user_id, 'highest_wallet', 0 );
        update_user_meta( $user_id, 'highest_profit', 0 );
        update_user_meta( $user_id, 'is_highest_agent', 'yes' );
        update_user_meta( $user_id, 'highest_sms_disabled', 0 );

        // Approval logic
        $needs_approval = get_option( 'highest_agent_registration_approval', 'no' ) === 'yes';
        if ( $needs_approval ) {
            update_user_meta( $user_id, 'highest_agent_approved', 'no' );
            $approval_msg = ' Your account will be reviewed and you will receive an SMS when approved.';
        } else {
            update_user_meta( $user_id, 'highest_agent_approved', 'yes' );
            $approval_msg = '';
            wp_set_auth_cookie( $user_id, true );
            wp_set_current_user( $user_id );
        }

        // Referral relationship
        if ( get_option( 'highest_enable_referral_system', 'no' ) === 'yes' && $referrer_id > 0 && isset( $referrer ) ) {
            global $wpdb;
            $wpdb->insert( $wpdb->prefix . 'highest_agent_referrals', array(
                'parent_agent_id' => $referrer->ID,
                'child_agent_id'  => $user_id,
                'bonus_type'      => 'percentage',
                'bonus_value'     => 0,
                'status'          => 'active',
            ) );
            $parent_phone = get_user_meta( $referrer->ID, 'phone', true );
            if ( $parent_phone ) {
                $this->send_sms( $parent_phone, "You have a new sub-agent! View them in your Referrals section.", 'welcome_agent' );
            }
        }

        // Welcome SMS
        if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) ) {
            if ( ! empty( $phone ) ) {
                $this->send_templated_sms( $phone, 'welcome_agent', array(
                    'name'      => $name,
                    'login_url' => home_url( '/agent-portal/' ),
                ) );
            }
        }

        // Notify admin of pending approval
        if ( $needs_approval ) {
            $admin_phone = get_option( 'highest_emergency_sms_phone', '' );
            if ( $admin_phone ) {
                $this->send_sms( $admin_phone, "New agent pending approval: {$name}, phone {$phone}, email {$email}", 'emergency_admin' );
            }
            wp_send_json_success( '🎉 Registration submitted! Admin will review your account.' );
        }

        wp_send_json_success( '🎉 Account created successfully! Redirecting to your Agent Portal...' );
    }

    public function ajax_save_agent_prices() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Please login.' );
        $user_id = get_current_user_id();
        if ( get_user_meta( $user_id, 'is_highest_agent', true ) !== 'yes' ) wp_send_json_error( 'Unauthorized access.' );
        $prices = get_user_meta( $user_id, 'highest_agent_prices', true ) ?: array();
        $has_error = false;
        $updated_count = 0;
        foreach ( $_POST as $key => $val ) {
            if ( strpos( $key, '-' ) !== false && is_numeric( $val ) ) {
                $val = floatval( $val );
                $clean_key = sanitize_text_field( $key );
                // Use the agent's ACTUAL cost as the floor: wholesale price (if set by parent)
                // takes priority over tier-based cost, which takes priority over the default.
                $wholesale_check = $this->get_subagent_wholesale_price( $user_id, $clean_key );
                $floor = $wholesale_check ? $wholesale_check['price'] : $this->get_agent_cost( $user_id, $clean_key );
                if ( $floor <= 0 ) {
                    $default_prices_ref = $this->get_default_agent_prices();
                    $floor = $default_prices_ref[ $clean_key ] ?? 0;
                }
                if ( $floor > 0 && $val < $floor + 0.1 ) { $has_error = true; continue; }
                $prices[ $clean_key ] = $val;
                $updated_count++;
            }
        }
        update_user_meta( $user_id, 'highest_agent_prices', $prices );
        if ( $has_error && $updated_count === 0 ) wp_send_json_error( 'You can only set prices strictly above your assigned agent price (add at least 0.1). No prices were saved.' );
        elseif ( $has_error ) wp_send_json_success( '✅ Some prices saved (prices below minimum were skipped).' );
        else wp_send_json_success( '✅ Prices saved successfully!' );
    }

    public function ajax_save_storefront() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Please login.' );
        $user_id = get_current_user_id();
        if ( get_user_meta( $user_id, 'is_highest_agent', true ) !== 'yes' ) wp_send_json_error( 'Unauthorized access.' );
        $name = sanitize_text_field( $_POST['store_name'] ?? '' );
        $whatsapp = sanitize_text_field( $_POST['store_whatsapp'] ?? '' );
        if ( empty( $name ) ) wp_send_json_error( 'Store name required.' );
        update_user_meta( $user_id, 'highest_store_name', $name );
        if ( $whatsapp ) update_user_meta( $user_id, 'highest_store_whatsapp', $whatsapp );
        if ( isset( $_POST['theme_color'] ) ) {
            update_user_meta( $user_id, 'highest_store_theme_color', sanitize_text_field( $_POST['theme_color'] ) );
        }
        if ( isset( $_POST['banner_url'] ) ) {
            update_user_meta( $user_id, 'highest_store_banner_url', esc_url_raw( $_POST['banner_url'] ) );
        }
        $store_url = home_url( '/store/' . sanitize_title( $name ) );
        wp_send_json_success( array(
            'message' => '✅ Store name updated!',
            'store_url' => $store_url
        ) );
    }

    public function ajax_agent_buy_data() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Please login.' );
        $user_id = get_current_user_id();
        if ( get_user_meta( $user_id, 'is_highest_agent', true ) !== 'yes' ) wp_send_json_error( 'Unauthorized.' );

        // Suspension check
        if ( get_user_meta( $user_id, 'highest_agent_suspended', true ) === '1' ) {
            wp_send_json_error( 'Your account is suspended. Contact admin.' );
        }

        $package_key       = sanitize_text_field( $_POST['package_key'] ?? '' );
        $phone             = sanitize_text_field( $_POST['phone'] ?? '' );
        $customer_price    = floatval( $_POST['selling_price'] ?? 0 );
        $default_prices    = $this->get_default_agent_prices();

        // Determine cost: check wholesale first, then tier pricing
        $wholesale_info = $this->get_subagent_wholesale_price( $user_id, $package_key );
        if ( $wholesale_info ) {
            $admin_agent_price = $wholesale_info['price'];
            $subagent_parent_id = $wholesale_info['parent_id'];
        } else {
            $admin_agent_price = $this->get_agent_cost( $user_id, $package_key );
            $subagent_parent_id = null;
        }

        // ── TEMPORARY DEBUG – remove after confirming wholesale works ──────────
        error_log( 'WHOLESALE DEBUG: subagent=' . $user_id . ' pkg=' . $package_key
            . ' wholesale_result=' . print_r( $wholesale_info, true ) );
        error_log( 'WHOLESALE DEBUG: final cost=' . $admin_agent_price
            . ' subagent_parent_id=' . var_export( $subagent_parent_id, true ) );
        // ── END DEBUG ──────────────────────────────────────────────────────────

        if ( $admin_agent_price <= 0 ) wp_send_json_error( 'Invalid package.' );
        if ( $customer_price < $admin_agent_price ) {
            wp_send_json_error( 'You cannot sell below your assigned agent price of GH₵' . number_format( $admin_agent_price, 2 ) );
        }

        $payment_method = sanitize_text_field( $_POST['payment_method'] ?? 'paystack' );
        $ref            = 'AGENT-' . time() . '-' . rand( 10000, 99999 );
        $network_id     = intval( $_POST['network_id'] ?? 3 );
        $shared_bundle  = intval( $_POST['shared_bundle'] ?? 1 );
        $network_name   = $this->get_network_name( $network_id );
        $package_name   = $network_name . '-' . $shared_bundle . 'GB';

        global $wpdb;

        // ── Wallet: deduct and log BEFORE calling DataKazina ────────────────
        // This guarantees the deduction is always recorded, even if the API
        // call hangs or returns an ambiguous response.
        if ( $payment_method === 'wallet' ) {
            $wallet = (float) get_user_meta( $user_id, 'highest_wallet', true );
            if ( $wallet < $customer_price ) {
                wp_send_json_error( 'Insufficient wallet balance. Needed GH₵' . number_format( $customer_price, 2 ) );
            }
            $new_wallet_balance = $wallet - $customer_price;
            update_user_meta( $user_id, 'highest_wallet', $new_wallet_balance );

            // Immediately log the full deduction (selling price, not just cost) –
            // always recorded regardless of API outcome. Profit is pre-funded here
            // and will be recorded as withdrawable below.
            $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                'user_id'       => $user_id,
                'amount'        => -$customer_price,
                'type'          => 'wallet_purchase',
                'status'        => 'approved',
                'reference'     => $ref,
                'balance_after' => $new_wallet_balance,
                'created_at'    => current_time( 'mysql' ),
            ) );
        }

        // ── Call DataKazina ──────────────────────────────────────────────────
        $payload = array(
            'recipient_msisdn' => $phone,
            'network_id'       => $network_id,
            'shared_bundle'    => $shared_bundle,
            'incoming_api_ref' => $ref,
        );
        $result = $this->dakazina_api_call( 'buy-data-package', 'POST', $payload );

        // Extract DataKazina reference before any failure checks
        $dakazina_ref = $this->deep_find_ref( $result['data'] ?? array() );
        $tracking_ref = ( ! empty( $dakazina_ref ) && $dakazina_ref !== $ref ) ? $dakazina_ref : $ref;

        // Detect known failure conditions
        $is_not_available = isset( $result['data']['message'] ) && stripos( $result['data']['message'], 'not available' ) !== false;
        $is_explicit_fail = isset( $result['data']['success'] ) && $result['data']['success'] === false;

        if ( $is_not_available || $is_explicit_fail ) {
            if ( $payment_method === 'wallet' ) {

                // ── Layer 1: Local duplicate check ───────────────────────────
                $local_check = $wpdb->get_var( $wpdb->prepare(
                    "SELECT id FROM {$wpdb->prefix}highest_orders
                     WHERE phone = %s
                       AND package = %s
                       AND selling_price = %s
                       AND created_at > DATE_SUB(NOW(), INTERVAL 5 MINUTE)
                       AND status != 'FAILED'
                     LIMIT 1",
                    $this->normalise_phone( $phone ),
                    $package_name,
                    $customer_price
                ) );
                if ( $local_check ) {
                    // Order already exists – do NOT refund
                    wp_send_json_error( $result['data']['message'] ?? 'Order already exists in system.' );
                }

                // ── Layer 2: Verify with DataKazina ──────────────────────────
                $check_ref     = ! empty( $dakazina_ref ) ? $dakazina_ref : $ref;
                $verify_result = $this->dakazina_api_call( 'fetch-single-transaction', 'POST', array( 'transaction_id' => $check_ref ) );

                $terminal_success = false;
                $terminal_failure = false;

                if ( ! isset( $verify_result['error'] ) ) {
                    $raw_status       = strtoupper( $this->deep_find_status( $verify_result['data'] ?? array() ) );
                    $terminal_success = stripos( $raw_status, 'DELIVER' ) !== false || stripos( $raw_status, 'SUCCESS' ) !== false;
                    $terminal_failure = stripos( $raw_status, 'FAIL' ) !== false
                                     || stripos( $raw_status, 'REJECT' ) !== false
                                     || stripos( $raw_status, 'CANCEL' ) !== false;
                }

                if ( $terminal_success ) {
                    // DataKazina delivered despite error – record as DELIVERED, no refund
                    $wpdb->insert( $wpdb->prefix . 'highest_orders', array(
                        'order_type'       => 'agent',
                        'user_id'          => $user_id,
                        'phone'            => $this->normalise_phone( $phone ),
                        'package'          => $package_name,
                        'selling_price'    => $customer_price,
                        'agent_cost'       => $admin_agent_price,
                        'status'           => 'DELIVERED',
                        'ref'              => $ref,
                        'incoming_api_ref' => $tracking_ref,
                        'api_response'     => wp_json_encode( $result ),
                        'created_at'       => current_time( 'mysql' ),
                    ) );
                    $profit_earned = round( $customer_price - $admin_agent_price, 2 );
                    update_user_meta( $user_id, 'highest_profit', (float) get_user_meta( $user_id, 'highest_profit', true ) + $profit_earned );
                    $this->add_profit_history( $user_id, $profit_earned, 'profit_earned', $ref, "Agent sale to {$phone} (verified delivered)" );
                    if ( $this->emergency_sms_enabled && ! empty( $this->emergency_sms_phone ) ) {
                        $this->send_templated_sms( $this->emergency_sms_phone, 'emergency_admin', array(
                            'ref'      => $ref,
                            'package'  => $package_name,
                            'phone'    => $phone,
                            'amount'   => number_format( $customer_price, 2 ),
                            'agent_id' => $user_id,
                        ) );
                    }
                    wp_send_json_error( $result['data']['message'] ?? 'DataKazina reported failure but order was delivered.' );

                } elseif ( $terminal_failure ) {
                    // Confirmed failure – refund wallet
                    $wallet     = (float) get_user_meta( $user_id, 'highest_wallet', true );
                    $new_wallet = $wallet + $admin_agent_price;
                    update_user_meta( $user_id, 'highest_wallet', $new_wallet );
                    $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                        'user_id'       => $user_id,
                        'amount'        => $admin_agent_price,
                        'type'          => 'wallet_purchase_refund',
                        'status'        => 'approved',
                        'reference'     => $ref . '-refund',
                        'balance_after' => $new_wallet,
                        'created_at'    => current_time( 'mysql' ),
                    ) );
                    $wpdb->insert( $wpdb->prefix . 'highest_orders', array(
                        'order_type'       => 'agent',
                        'user_id'          => $user_id,
                        'phone'            => $this->normalise_phone( $phone ),
                        'package'          => $package_name,
                        'selling_price'    => $customer_price,
                        'agent_cost'       => $admin_agent_price,
                        'status'           => 'FAILED',
                        'ref'              => $ref,
                        'incoming_api_ref' => $tracking_ref,
                        'api_response'     => wp_json_encode( $result ),
                        'created_at'       => current_time( 'mysql' ),
                    ) );
                    wp_send_json_error( $result['data']['message'] ?? 'DataKazina could not process the order. Wallet refunded.' );

                } else {
                    // ── UNCERTAIN: admin must review ─────────────────────────
                    // Wallet stays debited (already logged above).
                    // Order is flagged UNCERTAIN so admin can Resubmit, Refund, or Mark Delivered.
                    $wpdb->insert( $wpdb->prefix . 'highest_orders', array(
                        'order_type'       => 'agent',
                        'user_id'          => $user_id,
                        'phone'            => $this->normalise_phone( $phone ),
                        'package'          => $package_name,
                        'selling_price'    => $customer_price,
                        'agent_cost'       => $admin_agent_price,
                        'status'           => 'UNCERTAIN',
                        'ref'              => $ref,
                        'incoming_api_ref' => $tracking_ref,
                        'api_response'     => wp_json_encode( $result ),
                        'created_at'       => current_time( 'mysql' ),
                    ) );
                    wp_send_json_error( 'Order status is uncertain. Admin will review and resolve this order.' );
                }

            } else {
                // Paystack payment – no wallet involved
                wp_send_json_error( $result['data']['message'] ?? 'DataKazina could not process the order.' );
            }
        }

        // ── Success path ────────────────────────────────────────────────────
        $wpdb->insert( $wpdb->prefix . 'highest_orders', array(
            'order_type'       => 'agent',
            'user_id'          => $user_id,
            'phone'            => $this->normalise_phone( $phone ),
            'package'          => $package_name,
            'selling_price'    => $customer_price,
            'agent_cost'       => $admin_agent_price,
            'status'           => 'PLACED',
            'ref'              => $ref,
            'incoming_api_ref' => $tracking_ref,
            'api_response'     => wp_json_encode( $result ),
            'created_at'       => current_time( 'mysql' ),
        ) );
        $order_id = $wpdb->insert_id; // capture ID for referral earnings record

        // Note: wallet deduction was already logged before the API call above.
        // No second log needed here.

        $profit_earned  = round( $customer_price - $admin_agent_price, 2 );
        $current_profit = (float) get_user_meta( $user_id, 'highest_profit', true );
        update_user_meta( $user_id, 'highest_profit', $current_profit + $profit_earned );
        $this->add_profit_history( $user_id, $profit_earned, 'profit_earned', $ref, "Agent sale to {$phone}" );

        // Referral bonus (locked) or wholesale parent profit
        if ( get_option( 'highest_enable_referral_system', 'no' ) === 'yes' && $profit_earned > 0 ) {
            if ( $subagent_parent_id ) {
                // Wholesale pricing active – parent earns wholesale margin
                $parent_tier_cost = $this->get_agent_cost( $subagent_parent_id, $package_key );
                $parent_profit = round( $admin_agent_price - $parent_tier_cost, 2 );
                if ( $parent_profit > 0 ) {
                    if ( get_option( 'highest_enable_referral_maturation', 'yes' ) === 'yes' ) {
                        $wpdb->insert( $wpdb->prefix . 'highest_referral_locked', array(
                            'parent_agent_id' => $subagent_parent_id,
                            'child_agent_id'  => $user_id,
                            'order_id'        => $order_id,
                            'bonus_amount'    => $parent_profit,
                            'status'          => 'locked',
                            'created_at'      => current_time( 'mysql' ),
                        ) );
                    } else {
                        $parent_current = (float) get_user_meta( $subagent_parent_id, 'highest_profit', true );
                        update_user_meta( $subagent_parent_id, 'highest_profit', $parent_current + $parent_profit );
                        $this->add_profit_history( $subagent_parent_id, $parent_profit, 'wholesale_profit', $ref, "Wholesale profit from sub-agent sale" );
                    }
                }
            } else {
                // Percentage-based referral bonus
                $referral = $wpdb->get_row( $wpdb->prepare(
                    "SELECT parent_agent_id, bonus_value FROM {$wpdb->prefix}highest_agent_referrals
                     WHERE child_agent_id = %d
                       AND COALESCE( status, 'active' ) = 'active'", $user_id
                ) );
                if ( $referral ) {
                    $pct   = $referral->bonus_value > 0 ? intval( $referral->bonus_value ) : intval( get_user_meta( $referral->parent_agent_id, 'highest_referral_bonus_pct', true ) );
                    $limit = intval( get_option( 'highest_referral_bonus_limit_pct', 20 ) );
                    if ( $limit > 0 && $pct > $limit ) $pct = $limit;
                    if ( $pct > 0 ) {
                        $bonus  = round( $profit_earned * ( $pct / 100 ), 2 );
                        if ( get_option( 'highest_enable_referral_maturation', 'yes' ) === 'yes' ) {
                            $wpdb->insert( $wpdb->prefix . 'highest_referral_locked', array(
                                'parent_agent_id' => $referral->parent_agent_id,
                                'child_agent_id'  => $user_id,
                                'order_id'        => $order_id,
                                'bonus_amount'    => $bonus,
                                'status'          => 'locked',
                                'created_at'      => current_time( 'mysql' ),
                            ) );
                        } else {
                            $parent_profit = (float) get_user_meta( $referral->parent_agent_id, 'highest_profit', true );
                            update_user_meta( $referral->parent_agent_id, 'highest_profit', $parent_profit + $bonus );
                            $this->add_profit_history( $referral->parent_agent_id, $bonus, 'referral_bonus_percentage', 'ref-' . $user_id, "{$pct}% referral bonus from sub-agent sale" );
                        }
                    }
                }
            }
        }

        $agent_store_url = null;
        $store_name = get_user_meta( $user_id, 'highest_store_name', true );
        if ( $store_name ) $agent_store_url = home_url( '/store/' . sanitize_title( $store_name ) );
        $this->send_order_received_sms( $phone, $package_name, $ref, $agent_store_url );

        // NOTE: 'delivery_agent' SMS is intentionally NOT sent here.
        // The agent receives a delivery confirmation only when the order status
        // actually becomes DELIVERED (handled in update_order_status_by_ref).
        // Instead, send a lightweight "order placed" notification so the agent
        // knows the order was submitted successfully.
        if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) ) {
            $agent_phone = $this->get_agent_phone( $user_id );
            if ( ! empty( $agent_phone ) ) {
                $this->send_templated_sms( $agent_phone, 'order_placed_agent', array(
                    'package' => $package_name,
                    'phone'   => $phone,
                ) );
            }
        }

        $customer_email = sanitize_email( $_POST['email'] ?? '' );
        if ( ! empty( $customer_email ) ) {
            $this->send_customer_receipt( $customer_email, $phone, $package_name, $customer_price, $ref );
        }
        wp_send_json_success( 'Data purchased successfully' );
    }


    public function ajax_get_orders() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Please login.' );
        $user_id = get_current_user_id();
        global $wpdb;
        $orders = $wpdb->get_results( $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}highest_orders WHERE user_id = %d ORDER BY created_at DESC", $user_id ) );
        wp_send_json_success( $orders );
    }

    public function ajax_track_order() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Please login.' );
        $phone = sanitize_text_field( $_POST['phone'] ?? '' );
        $user_id = get_current_user_id();
        global $wpdb;
        $orders = $wpdb->get_results( $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}highest_orders WHERE user_id = %d AND phone LIKE %s ORDER BY created_at DESC", $user_id, '%' . $wpdb->esc_like( $phone ) . '%' ) );
        wp_send_json_success( $orders );
    }

    public function ajax_public_track_order() {
    $this->verify_ajax_request();
    $phone = sanitize_text_field( $_POST['phone'] ?? '' );
    if ( empty( $phone ) ) wp_send_json_error( 'Phone number is required.' );
    global $wpdb;
    $orders = $wpdb->get_results( $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}highest_orders WHERE phone LIKE %s ORDER BY created_at DESC", '%' . $wpdb->esc_like( $phone ) . '%' ) );
    foreach ( $orders as &$order ) {
        if ( strtoupper( $order->status ) === 'PROCESSING' ) {
            $check_ref = ! empty( $order->incoming_api_ref ) ? $order->incoming_api_ref : $order->ref;
            $result = $this->dakazina_api_call( 'fetch-single-transaction', 'POST', array( 'transaction_id' => $check_ref ) );
            if ( ! isset( $result['error'] ) ) {
                $raw_status = $this->deep_find_status( $result['data'] ?? [] );
                if ( $raw_status !== '' ) {
                    // Keep raw status – no normalisation
                    $new_status = strtoupper( trim( $raw_status ) ) ?: 'PROCESSING';
                    $this->update_order_status_by_ref( $check_ref, $new_status, array( 'api_response' => wp_json_encode( $result ) ) );
                    $order->status = $new_status;
                }
            }
            usleep( 150000 );
        }
    }
    unset( $order );
    wp_send_json_success( $orders );
}

    public function ajax_check_dakazina_status() {
    $this->verify_ajax_request();
    $ref = sanitize_text_field( $_POST['ref'] ?? '' );
    if ( empty( $ref ) ) wp_send_json_error( 'Reference is required.' );
    $result = $this->dakazina_api_call( 'fetch-single-transaction', 'POST', array( 'transaction_id' => $ref ) );
    if ( isset( $result['error'] ) ) wp_send_json_error( $result['error'] );
    $raw_status = $this->deep_find_status( $result['data'] ?? [] );
    $new_status = strtoupper( trim( $raw_status ?: '' ) ) ?: 'PROCESSING';
    $this->update_order_status_by_ref( $ref, $new_status, array( 'api_response' => wp_json_encode( $result ) ) );
    wp_send_json_success( array( 'status' => $new_status, 'api_response' => $result ) );
}

    public function ajax_auto_check_statuses() {
    $this->verify_ajax_request();
    global $wpdb;
    $table = $wpdb->prefix . 'highest_orders';
    $phone = sanitize_text_field( $_POST['phone'] ?? '' );
    $user_id = is_user_logged_in() ? get_current_user_id() : 0;
    if ( $phone ) {
        $orders = $wpdb->get_results( $wpdb->prepare( "SELECT ref, incoming_api_ref FROM $table WHERE phone LIKE %s AND (status IN ('PROCESSING', 'processing', 'PLACED', 'placed')) ORDER BY created_at DESC LIMIT 8", '%' . $wpdb->esc_like( $phone ) . '%' ) );
    } elseif ( $user_id ) {
        $orders = $wpdb->get_results( $wpdb->prepare( "SELECT ref, incoming_api_ref FROM $table WHERE user_id = %d AND (status IN ('PROCESSING', 'processing', 'PLACED', 'placed')) ORDER BY created_at DESC LIMIT 8", $user_id ) );
    } else {
        wp_send_json_success( [] );
        return;
    }

    $updated = [];
    foreach ( $orders as $order ) {
        $check_ref = ! empty( $order->incoming_api_ref ) ? $order->incoming_api_ref : $order->ref;
        $result = $this->dakazina_api_call( 'fetch-single-transaction', 'POST', array( 'transaction_id' => $check_ref ) );
        if ( ! isset( $result['error'] ) ) {
            $raw_status = $this->deep_find_status( $result['data'] ?? [] );
            if ( $raw_status !== '' ) {
                $new_status = strtoupper( trim( $raw_status ) ) ?: 'PROCESSING';
                $this->update_order_status_by_ref( $check_ref, $new_status, array( 'api_response' => wp_json_encode( $result ) ) );
                $updated[ $order->ref ] = $new_status;
            }
        }
        usleep( 150000 );
    }
    wp_send_json_success( $updated );
}

    public function ajax_request_manual_topup() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Please login.' );
        $user_id = get_current_user_id();
        if ( get_user_meta( $user_id, 'is_highest_agent', true ) !== 'yes' ) wp_send_json_error( 'Unauthorized.' );
        $amount = floatval( $_POST['amount'] ?? 0 );
        if ( $amount <= 0 ) wp_send_json_error( 'Invalid amount.' );
        $requests = get_user_meta( $user_id, 'highest_manual_topup_requests', true ) ?: array();
        $requests[] = array( 'id' => 'mt_' . uniqid(), 'amount' => $amount, 'status' => 'pending', 'date' => current_time( 'mysql' ) );
        update_user_meta( $user_id, 'highest_manual_topup_requests', $requests );
        wp_send_json_success( 'Request sent to admin.' );
    }

    public function ajax_update_profile() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Please login.' );
        $user_id = get_current_user_id();
        if ( get_user_meta( $user_id, 'is_highest_agent', true ) !== 'yes' ) wp_send_json_error( 'Unauthorized.' );
        $name = sanitize_text_field( $_POST['name'] ?? '' );
        $phone = sanitize_text_field( $_POST['phone'] ?? '' );
        $email = sanitize_email( $_POST['email'] ?? '' );
        $password = sanitize_text_field( $_POST['password'] ?? '' );
        $sms_disabled = isset( $_POST['sms_disabled'] ) ? (bool) $_POST['sms_disabled'] : false;

        if ( empty( $email ) || ! is_email( $email ) ) wp_send_json_error( 'Valid email required.' );
        if ( email_exists( $email ) && email_exists( $email ) != $user_id ) wp_send_json_error( 'Email already in use by another account.' );

        wp_update_user( array( 'ID' => $user_id, 'display_name' => $name ?: '', 'first_name' => $name ?: '', 'user_email' => $email ) );
        update_user_meta( $user_id, 'phone', $phone );
        update_user_meta( $user_id, 'highest_sms_disabled', $sms_disabled ? 1 : 0 );
        update_user_meta( $user_id, 'highest_momo_provider', sanitize_text_field( $_POST['momo_provider'] ?? '' ) );
        update_user_meta( $user_id, 'highest_momo_phone', sanitize_text_field( $_POST['momo_phone'] ?? '' ) );
        if ( ! empty( $password ) ) { wp_set_password( $password, $user_id ); wp_set_auth_cookie( $user_id ); }
        wp_send_json_success( 'Profile updated successfully.' );
    }

    public function ajax_verify_topup() {
        $this->verify_ajax_request();
        if ( ! is_user_logged_in() ) wp_send_json_error( 'Session expired. Please log in and try again.' );
        $ref = sanitize_text_field( $_POST['reference'] ?? '' );
        if ( empty( $ref ) ) wp_send_json_error( 'No payment reference provided.' );
        $secret_key = get_option( 'highest_paystack_secret_key', '' );
        if ( empty( $secret_key ) ) wp_send_json_error( 'Paystack secret key not configured. Contact admin.' );

        $response = wp_remote_get( 'https://api.paystack.co/transaction/verify/' . urlencode( $ref ), array(
            'headers' => array( 'Authorization' => 'Bearer ' . $secret_key ),
            'timeout' => 20,
        ) );
        if ( is_wp_error( $response ) ) wp_send_json_error( 'Paystack API request failed: ' . $response->get_error_message() );

        $body = json_decode( wp_remote_retrieve_body( $response ), true );
        if ( isset( $body['status'] ) && $body['status'] && isset( $body['data']['status'] ) && $body['data']['status'] === 'success' ) {
            $amount_ghs = floatval( $body['data']['amount'] ) / 100;
            $user_id = get_current_user_id();

            $processed = get_user_meta( $user_id, 'highest_processed_topups', true ) ?: array();
            if ( in_array( $ref, $processed ) ) wp_send_json_error( 'This transaction has already been processed.' );

            $current_wallet = (float) get_user_meta( $user_id, 'highest_wallet', true );
            $new_wallet = $current_wallet + $amount_ghs;
            update_user_meta( $user_id, 'highest_wallet', $new_wallet );

            $processed[] = $ref;
            update_user_meta( $user_id, 'highest_processed_topups', $processed );

            global $wpdb;
            $wpdb->insert( $wpdb->prefix . 'highest_topup_history', array(
                'user_id'   => $user_id,
                'amount'    => $amount_ghs,
                'type'      => 'paystack',
                'status'    => 'approved',
                'reference' => $ref,
                'created_at'=> current_time( 'mysql' ),
            ) );
            $wpdb->update( $wpdb->prefix . 'highest_topup_history', array( 'balance_after' => $new_wallet ), array( 'id' => $wpdb->insert_id ) );

            if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) ) {
                $agent_phone = $this->get_agent_phone( $user_id );
                if ( ! empty( $agent_phone ) ) {
                    $this->send_templated_sms( $agent_phone, 'wallet_topup', array(
                        'amount'  => number_format( $amount_ghs, 2 ),
                        'balance' => number_format( $new_wallet, 2 ),
                    ) );
                }
            }
            wp_send_json_success( 'Wallet topped up successfully by GH₵' . number_format( $amount_ghs, 2 ) );
        }
        $paystack_status = $body['data']['status'] ?? 'unknown';
        wp_send_json_error( 'Payment verification failed. Paystack status: ' . $paystack_status . '. If you were charged, contact support with ref: ' . $ref );
    }

    public function handle_agent_profit_withdrawal() {
        if ( ! isset( $_POST['hdp_withdraw_btn'] ) || ! is_user_logged_in() ) return;
        $user_id = get_current_user_id();

        // Suspension check
        if ( get_user_meta( $user_id, 'highest_agent_suspended', true ) === '1' ) {
            wp_redirect( add_query_arg( 'withdraw', 'suspended', wp_get_referer() ) );
            exit;
        }

        $amount  = (float) ( $_POST['withdraw_amount'] ?? 0 );
        $method  = sanitize_text_field( $_POST['withdrawal_method'] ?? 'manual' );
        $current_profit = (float) get_user_meta( $user_id, 'highest_profit', true );

        if ( $amount <= 0 || $amount > $current_profit ) {
            wp_redirect( add_query_arg( 'withdraw', 'failed_balance', wp_get_referer() ) );
            exit;
        }

        // Deduct profit immediately and log
        $new_profit = $current_profit - $amount;
        update_user_meta( $user_id, 'highest_profit', $new_profit );
        $this->add_profit_history( $user_id, -$amount, 'withdrawal_request', 'WD-' . uniqid(), "Withdrawal request via {$method}" );

        $fee        = 0;
        $net_amount = $amount;

        if ( $method === 'auto' ) {
            if ( $amount < 80 ) {
                $fee = 1.00;
            } else {
                $fee = round( $amount * 0.0198, 2 );
            }
            $net_amount = round( $amount - $fee, 2 );

            $provider = sanitize_text_field( $_POST['momo_provider'] ?? '' );
            $phone    = sanitize_text_field( $_POST['momo_phone'] ?? '' );

            if ( empty( $provider ) || empty( $phone ) ) {
                // Refund
                update_user_meta( $user_id, 'highest_profit', $current_profit );
                $this->add_profit_history( $user_id, $amount, 'withdrawal_refund', 'WD-' . uniqid(), 'Refund – missing MoMo details' );
                wp_redirect( add_query_arg( 'withdraw', 'no_momo', wp_get_referer() ) );
                exit;
            }

            // If admin has disabled approval requirement, process instantly
            if ( get_option( 'highest_withdrawal_requires_approval', 'yes' ) === 'no' ) {
                $transfer_result = $this->process_auto_withdrawal( $user_id, $provider, $phone, $net_amount );
                if ( is_wp_error( $transfer_result ) ) {
                    // Transfer failed – refund profit
                    update_user_meta( $user_id, 'highest_profit', $current_profit );
                    $this->add_profit_history( $user_id, $amount, 'withdrawal_refund', 'WD-' . uniqid(), 'Refund – MoMo transfer failed: ' . $transfer_result->get_error_message() );
                    wp_redirect( add_query_arg( 'withdraw', 'transfer_failed', wp_get_referer() ) );
                    exit;
                }
                // Transfer successful – record as completed
                $requests   = get_user_meta( $user_id, 'highest_withdraw_requests', true ) ?: array();
                $requests[] = array(
                    'id'            => 'wd_' . uniqid(),
                    'amount'        => $amount,
                    'fee'           => $fee,
                    'net_amount'    => $net_amount,
                    'method'        => 'auto',
                    'provider'      => $provider,
                    'phone'         => $phone,
                    'status'        => 'completed',
                    'transfer_code' => $transfer_result,
                    'date'          => current_time( 'mysql' ),
                );
                update_user_meta( $user_id, 'highest_withdraw_requests', $requests );

                // SMS notification
                if ( $this->is_sms_enabled() && ! $this->is_agent_sms_disabled( $user_id ) ) {
                    $agent_phone = $this->get_agent_phone( $user_id );
                    if ( ! empty( $agent_phone ) ) {
                        $this->send_templated_sms( $agent_phone, 'withdrawal_update', array(
                            'amount' => number_format( $net_amount, 2 ),
                        ) );
                    }
                }
                wp_redirect( add_query_arg( 'withdraw', 'instant_success', wp_get_referer() ) );
                exit;
            }

            // Approval required – save as pending
            $requests   = get_user_meta( $user_id, 'highest_withdraw_requests', true ) ?: array();
            $requests[] = array(
                'id'         => 'wd_' . uniqid(),
                'amount'     => $amount,
                'fee'        => $fee,
                'net_amount' => $net_amount,
                'method'     => 'auto',
                'provider'   => $provider,
                'phone'      => $phone,
                'status'     => 'pending_approval',
                'date'       => current_time( 'mysql' ),
            );
            update_user_meta( $user_id, 'highest_withdraw_requests', $requests );

        } else {
            // Manual bank transfer – ALWAYS requires admin approval regardless of toggle
            $account = sanitize_text_field( $_POST['account_number'] ?? '' );

            if ( empty( $account ) ) {
                update_user_meta( $user_id, 'highest_profit', $current_profit );
                $this->add_profit_history( $user_id, $amount, 'withdrawal_refund', 'WD-' . uniqid(), 'Refund – missing account number' );
                wp_redirect( add_query_arg( 'withdraw', 'no_account', wp_get_referer() ) );
                exit;
            }

            $requests   = get_user_meta( $user_id, 'highest_withdraw_requests', true ) ?: array();
            $requests[] = array(
                'id'         => 'wd_' . uniqid(),
                'amount'     => $amount,
                'fee'        => 0,
                'net_amount' => $amount,
                'method'     => 'manual',
                'account'    => $account,
                'status'     => 'pending_approval', // Manual withdrawals ALWAYS require admin approval
                'date'       => current_time( 'mysql' ),
            );
            update_user_meta( $user_id, 'highest_withdraw_requests', $requests );
        }

        wp_redirect( add_query_arg( 'withdraw', 'success', wp_get_referer() ) );
        exit;
    }

    public function render_agent_transactions( $user_id ) {
        global $wpdb;
        $txns = $wpdb->get_results( $wpdb->prepare( "SELECT * FROM {$wpdb->prefix}highest_orders WHERE user_id = %d ORDER BY created_at DESC", $user_id ) );
        if ( empty( $txns ) ) return '<p class="text-slate-500 italic">No transactions yet.</p>';
        $html = '<ul class="space-y-3">';
        foreach ( $txns as $txn ) {
            $date_formatted = date( 'd M Y • H:i', strtotime( $txn->created_at ) );
            $ref = $txn->ref ?? '—';
            $status_upper = strtoupper( $txn->status ?? 'PROCESSING' );
            $statusClass = $status_upper === 'DELIVERED' ? 'bg-emerald-100 text-emerald-700' : ( $status_upper === 'FAILED' ? 'bg-red-100 text-red-700' : 'bg-amber-100 text-amber-700' );
            $html .= '<li class="flex flex-col md:flex-row justify-between items-start md:items-center bg-slate-50 p-5 rounded-3xl text-sm gap-4">';
            $html .= '<div class="flex-1"><span class="font-semibold text-base">' . esc_html( strtoupper( $txn->package ?? 'DATA' ) ) . '</span>';
            $html .= '<div class="text-xs text-slate-400 mt-1">📱 ' . esc_html( $txn->phone ?? '—' ) . ' &nbsp;|&nbsp; Ref: <code>' . esc_html( $ref ) . '</code></div></div>';
            $html .= '<div class="text-right"><div class="text-xl font-bold text-emerald-700">GH₵' . number_format( $txn->selling_price ?? 0, 2 ) . '</div>';
            $html .= '<span id="status-' . esc_attr( $ref ) . '" class="inline-block mt-1 px-4 py-1 text-xs rounded-3xl ' . $statusClass . '">' . esc_html( $status_upper ) . '</span></div>';
            $html .= '<div class="text-xs text-slate-400 md:text-right">' . esc_html( $date_formatted ) . '</div>';
            // Live Check button removed – status is updated by cron only
            $html .= '</li>';
        }
        return $html . '</ul>';
    }

    public function create_agent_portal_page() {
        if ( ! get_page_by_path( 'agent-portal' ) ) {
            wp_insert_post( array(
                'post_title'   => 'Agent Portal',
                'post_name'    => 'agent-portal',
                'post_content' => '[highestdataplug_agent_portal]',
                'post_status'  => 'publish',
                'post_type'    => 'page',
            ) );
        }
    }

    public function redirect_agents_to_portal() {
        if ( ! is_user_logged_in() || is_admin() ) return;
        $user_id = get_current_user_id();
        if ( get_user_meta( $user_id, 'is_highest_agent', true ) !== 'yes' ) return;
        if ( is_page( 'agent-portal' ) ) return;
        $current_page = get_query_var( 'pagename' );
        if ( in_array( $current_page, array( 'login', 'register', 'wp-login' ), true ) ) return;
        wp_redirect( home_url( '/agent-portal/' ) );
        exit;
    }

    public function redirect_agent_login( $redirect_to, $request, $user ) {
        if ( isset( $user->roles ) && is_array( $user->roles ) && get_user_meta( $user->ID, 'is_highest_agent', true ) === 'yes' ) {
            return home_url( '/agent-portal/' );
        }
        return $redirect_to;
    }

    public function render_agent_login_form() {
        if ( is_user_logged_in() ) {
            return '<div class="max-w-md mx-auto text-center p-8 bg-white rounded-3xl shadow-sm border border-slate-100"><p class="mb-4 text-slate-600">You are already logged in.</p><a href="' . esc_url( home_url( '/agent-portal/' ) ) . '" class="inline-block bg-[#001F3F] text-[#FFD700] px-8 py-3 rounded-2xl font-bold shadow-md hover:scale-105 transition-transform">Go to Agent Portal</a></div>';
        }
        ob_start(); ?>
        <div class="max-w-md mx-auto bg-white p-8 rounded-[2.5rem] shadow-xl border border-slate-100 mt-10 mb-10">
            <div class="text-center mb-8"><i class="fas fa-user-shield text-5xl text-[#001F3F] mb-4"></i><h2 class="text-3xl font-bold text-slate-800">Agent Login</h2><p class="text-slate-500 mt-2 text-sm">Welcome back! Please log in to your account.</p></div>
            <form method="post" action="<?php echo esc_url( site_url( 'wp-login.php', 'login_post' ) ); ?>" class="space-y-5">
                <div><input type="text" name="log" placeholder="Username or Email" required class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl outline-none focus:border-[#001F3F] focus:ring-4 focus:ring-[#001F3F]/10 transition-all text-slate-700"></div>
                <div><input type="password" name="pwd" placeholder="Password" required class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl outline-none focus:border-[#001F3F] focus:ring-4 focus:ring-[#001F3F]/10 transition-all text-slate-700"></div>
                <div class="flex items-center gap-2 px-2 pb-2"><input name="rememberme" type="checkbox" id="agent-remember" value="forever" class="w-4 h-4 text-[#001F3F] rounded"><label for="agent-remember" class="text-sm text-slate-600 cursor-pointer select-none">Remember Me</label></div>
                <input type="hidden" name="redirect_to" value="<?php echo esc_url( home_url( '/agent-portal/' ) ); ?>">
                <button type="submit" name="wp-submit" class="w-full py-4 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-bold text-lg rounded-2xl transition-all shadow-lg hover:shadow-xl hover:-translate-y-0.5 flex items-center justify-center gap-2"><i class="fas fa-sign-in-alt"></i> Secure Login</button>
            </form>
            <div class="text-center mt-6"><p class="text-slate-500 text-sm">Don't have an account?</p><a href="<?php echo esc_url( home_url( '/' ) ); ?>" class="inline-flex items-center gap-2 px-6 py-3 bg-white border-2 border-[#FFD700] text-[#001F3F] font-semibold rounded-3xl hover:bg-[#FFD700] hover:text-[#001F3F] transition-all mt-3"><i class="fas fa-user-plus"></i> Sign Up Now</a></div>
        </div>
        <?php
        return ob_get_clean();
    }

    // ─── Fullsite (Public Buy) with Announcement ──────────
    public function render_fullsite() {
        ob_start();
        $login_url = home_url( '/agent-login/' );
        $nonce = wp_create_nonce( 'highest_register' );
        $disable_public_buy = get_option( 'highest_disable_public_buy', false );
        $disable_all_buy = get_option( 'highest_disable_all_buy', false );
        $disable_marketplace_banner = get_option( 'highest_disable_marketplace_banner', false );
        $whatsapp = get_option( 'highest_whatsapp_number', '0551907463' );
        $announce_enabled = get_option( 'highest_public_announcement_enabled', false );
        $announce_text    = get_option( 'highest_public_announcement_text', '' );
        ?>
        <div class="w-screen px-4 py-8 bg-gradient-to-br from-[#001F3F]/5 to-[#FFD700]/5 min-h-screen">
            <div class="flex items-center justify-between bg-[#001F3F] shadow-xl rounded-3xl px-6 py-4 mb-10 sticky top-0 z-50 text-white">
                <div class="flex items-center gap-4"><img src="https://highestdataplug.com/wp-content/uploads/2026/04/20ybve.jpg" alt="HIGHEST DATA PLUG" class="h-12 w-auto"><div><h1 class="text-3xl font-bold tracking-tighter text-[#FFD700]">HIGHEST DATA PLUG</h1><p class="text-[#FFD700] text-sm -mt-1">Poku Asare Ventures</p></div></div>
                <div class="flex items-center gap-4"><div class="relative group"><button onclick="toggleAgentMenu()" class="px-6 py-2 bg-white/10 hover:bg-white/20 text-white rounded-3xl text-sm font-medium transition flex itemscenter gap-2"><i class="fas fa-bars"></i> Agent Menu</button><div id="agent-menu-dropdown" class="hidden absolute right-0 mt-2 w-72 bg-white rounded-3xl shadow-2xl py-2 text-slate-800 z-[99999]"><a href="<?php echo esc_url( $login_url ); ?>" class="flex items-center gap-3 px-6 py-3 hover:bg-slate-100 border-b"><i class="fas fa-user"></i> Agent Login</a><button onclick="showRegisterModal();toggleAgentMenu()" class="flex items-center gap-3 w-full text-left px-6 py-3 hover:bg-slate-100 border-b"><i class="fas fa-user-plus"></i> Become an Agent</button><a href="https://chat.whatsapp.com/DXsDCF4y1nP1HKW2va87EW" target="_blank" class="flex items-center gap-3 px-6 py-4 hover:bg-emerald-50 text-emerald-600 font-semibold"><i class="fab fa-whatsapp text-2xl"></i><div>Join Our Community<br><span class="text-xs text-emerald-500">WhatsApp Group</span></div></a><a href="https://wa.me/<?php echo esc_attr( str_replace( '+', '', $whatsapp ) ); ?>?text=Hi%20Highest%20Data%20Plug%20support%2C%20I%20need%20help" target="_blank" class="flex items-center gap-3 px-6 py-3 hover:bg-slate-100 border-t text-[#25D366]"><i class="fab fa-whatsapp"></i> Customer Support Chat</a></div></div><a onclick="document.getElementById('public-tracker-section').scrollIntoView({behavior:'smooth'})" class="flex items-center gap-2 px-5 py-2 bg-white/10 hover:bg-white/20 text-white rounded-3xl text-sm font-medium transition cursor-pointer"><i class="fas fa-search"></i><span>Track Your Order</span></a> </div>
            </div>
            <?php if ( ! $disable_marketplace_banner ) : ?>
            <section class="mb-8 bg-gradient-to-r from-[#001F3F] to-[#FFD700] text-white p-6 rounded-3xl shadow-lg text-center">
                <h2 class="text-2xl font-bold mb-2">🚀 Wanna sell a product or find a job?</h2>
                <p class="mb-4 text-lg">Visit our Marketplace!</p>
                <a href="https://test.highestdataplug.com" target="_blank" class="inline-block bg-white text-[#001F3F] px-8 py-3 rounded-2xl font-bold shadow-md hover:scale-105 transition-transform">Visit Marketplace →</a>
            </section>
                    <?php endif; ?>

        <!-- 🔔 INSTALL APP BUTTON (PUBLIC) – always visible -->
<div class="text-center mb-8" id="pwa-install-container">
    <button id="pwa-install-btn-public" class="bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] px-8 py-4 rounded-2xl font-bold text-lg shadow-lg transition-transform"
            onclick="installPublic()">
        <i class="fas fa-download mr-2"></i> Install App for Quick Access
    </button>
</div>

        <?php if ( $announce_enabled && ! empty( $announce_text ) ) : ?>
        <section class="mb-8 bg-amber-50 border-2 border-amber-400 p-6 rounded-3xl shadow-md text-center">
            <div class="text-slate-800 font-bold text-xl mb-2 flex items-center justify-center gap-2">
                <i class="fas fa-bullhorn text-amber-600"></i> ANNOUNCEMENT
            </div>
            <p class="text-slate-700"><?php echo nl2br( esc_html( $announce_text ) ); ?></p>
        </section>
        <?php endif; ?>
        <?php if ( get_option( 'highest_show_whatsapp_banner', 'yes' ) === 'yes' ) : ?>
        <section class="mb-8 bg-gradient-to-r from-emerald-500 to-green-600 text-white p-6 rounded-3xl shadow-lg text-center">
            <h2 class="text-2xl font-bold mb-2 flex items-center justify-center gap-3">
                <i class="fab fa-whatsapp text-4xl"></i> Join Our WhatsApp Community
            </h2>
            <p class="mb-4 text-lg">Get instant updates, exclusive deals, and direct support from our team.</p>
            <a href="<?php echo esc_url( get_option( 'highest_community_link', 'https://chat.whatsapp.com/DXsDCF4y1nP1HKW2va87EW' ) ); ?>" target="_blank"
               class="inline-block bg-white text-emerald-700 px-8 py-3 rounded-2xl font-bold shadow-md hover:scale-105 transition-transform">
                Join Community Now →
            </a>
        </section>
        <?php endif; ?>
        <section class="bg-white rounded-[2.5rem] p-6 md:p-8 shadow-sm border border-slate-200"><h3 class="text-2xl font-bold mb-8 flex items-center justify-center gap-3 text-slate-800">⚡ Buy Data</h3><input id="walkin-phone" placeholder="0551234567" maxlength="10" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl mb-8 focus:ring-4 focus:ring-[#FFD700]/20 outline-none text-center"><div class="text-xs text-amber-600 -mt-6 mb-6 flex items-center gap-1"><i class="fas fa-info-circle"></i> Verify your number (10 digits only)</div><div id="walkin-grid" class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-3 md:gap-4"></div></section>
            <section id="public-tracker-section" class="mt-12 bg-gradient-to-br from-[#001F3F] to-[#FFD700]/10 rounded-3xl p-6 shadow-xl"><h2 class="text-xl font-semibold mb-4 flex items-center gap-3 text-[#001F3F]"><i class="fas fa-search"></i> Track Your Order (Live)</h2><?php if($tm=$this->get_delivery_message('highest_tracker_header_msg','Data delivery usually takes about {minutes} minute(s).'))echo'<p class="text-sm text-slate-500 -mt-2 mb-3">'.$tm.'</p>';?><div class="flex flex-col md:flex-row gap-3 max-w-lg"><input id="public-tracker-phone" placeholder="0551234567" maxlength="10" class="flex-1 p-4 border-2 border-[#FFD700] rounded-3xl text-base focus:outline-none focus:border-[#001F3F]"><button onclick="publicTrackOrder()" class="px-8 py-4 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-semibold rounded-3xl text-base">Track Now</button></div><div id="public-tracker-result" class="mt-6 bg-white rounded-3xl p-5 min-h-[180px]"></div><div class="text-xs text-amber-600 mt-3 flex items-center gap-1"><i class="fas fa-info-circle"></i> Verify your number (10 digits only)</div></section>
            <?php if ( ! $disable_marketplace_banner ) : ?>
            <section class="mt-12 bg-gradient-to-r from-[#001F3F] to-[#FFD700] text-white p-6 rounded-3xl shadow-lg text-center">
                <h2 class="text-2xl font-bold mb-2">🚀 Wanna sell a product or find a job?</h2>
                <p class="mb-4 text-lg">Visit our Marketplace!</p>
                <a href="https://test.highestdataplug.com" target="_blank" class="inline-block bg-white text-[#001F3F] px-8 py-3 rounded-2xl font-bold shadow-md hover:scale-105 transition-transform">Visit Marketplace →</a>
            </section>
            <?php endif; ?>
            <a href="https://wa.me/<?php echo esc_attr( str_replace( '+', '', $whatsapp ) ); ?>?text=Hi%20Highest%20Data%20Plug%20support%2C%20I%20need%20help" target="_blank" class="fixed bottom-6 right-6 bg-[#25D366] text-white w-14 h-14 rounded-3xl flex items-center justify-center shadow-2xl hover:scale-110 transition text-3xl z-[99999]"><i class="fab fa-whatsapp"></i></a>
        </div>
        <!-- Modals and Scripts -->
        <div id="register-modal" class="hidden fixed inset-0 bg-black/70 flex items-center justify-center z-[9999] p-4"><div class="bg-white rounded-3xl p-8 w-full max-w-md shadow-2xl"><div class="flex justify-between items-center mb-6"><h3 class="text-2xl font-semibold text-gray-800">Create Agent Account</h3><button onclick="closeRegisterModal()" class="text-gray-400 hover:text-gray-600 text-2xl">×</button></div><form id="register-form" class="space-y-5"><input type="text" id="reg-name" placeholder="Full Name" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]" required><input type="tel" id="reg-phone" placeholder="Phone Number (0551234567)" maxlength="10" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]" required><div class="text-xs text-amber-600 -mt-2 mb-2 flex items-center gap-1"><i class="fas fa-info-circle"></i> Verify your number (10 digits only)</div><input type="email" id="reg-email" placeholder="Email Address (for login)" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]" required><input type="password" id="reg-password" placeholder="Create Password (min 8 chars)" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]" required><button type="submit" id="submit-btn" class="w-full py-4 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-semibold rounded-3xl transition flex items-center justify-center">Create My Agent Account</button></form></div></div>
        <div id="buy-phone-modal" class="hidden fixed inset-0 bg-black/70 flex items-center justify-center z-[9999] p-4"><div class="bg-white rounded-3xl p-8 w-full max-w-md shadow-2xl"><div class="flex justify-between items-center mb-6"><h3 class="text-2xl font-semibold text-gray-800">Customer Telephone Number</h3><button onclick="closeBuyModal()" class="text-gray-400 hover:text-gray-600 text-2xl">×</button></div><div class="space-y-5"><input type="tel" id="modal-phone" placeholder="0551234567" maxlength="10" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F] text-lg" required><div class="text-xs text-amber-600 flex items-center gap-1"><i class="fas fa-info-circle"></i> Verify your Ghana number (10 digits only)</div><input type="email" id="modal-email" placeholder="Email (optional - for Paystack receipt)" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F] text-lg"><div class="text-xs text-amber-600 flex items-center gap-1"><i class="fas fa-exclamation-triangle"></i> ⚠️ Please enter your email to avoid payment failure.</div><button onclick="submitBuyPhone()" class="w-full py-4 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-semibold rounded-3xl transition flex items-center justify-center"><i class="fas fa-credit-card mr-2"></i> Continue to Secure Payment</button></div></div></div>
        <script>
        const bundles = [ { network:"MTN", size:"1GB", price:4.65, network_id:3, shared_bundle:1 }, { network:"MTN", size:"2GB", price:9.40, network_id:3, shared_bundle:2 }, { network:"MTN", size:"3GB", price:14.00, network_id:3, shared_bundle:3 }, { network:"MTN", size:"4GB", price:18.50, network_id:3, shared_bundle:4 }, { network:"MTN", size:"5GB", price:23.00, network_id:3, shared_bundle:5 }, { network:"MTN", size:"6GB", price:27.50, network_id:3, shared_bundle:6 }, { network:"MTN", size:"10GB", price:42.70, network_id:3, shared_bundle:10 }, { network:"MTN", size:"15GB", price:62.90, network_id:3, shared_bundle:15 }, { network:"MTN", size:"20GB", price:83.00, network_id:3, shared_bundle:20 }, { network:"MTN", size:"25GB", price:103.00, network_id:3, shared_bundle:25 }, { network:"MTN", size:"30GB", price:122.50, network_id:3, shared_bundle:30 }, { network:"MTN", size:"40GB", price:162.80, network_id:3, shared_bundle:40 }, { network:"MTN", size:"50GB", price:199.40, network_id:3, shared_bundle:50 }, { network:"MTN", size:"100GB", price:394.00, network_id:3, shared_bundle:100 }, { network:"TELECEL", size:"5GB", price:23.00, network_id:2, shared_bundle:5 }, { network:"TELECEL", size:"10GB", price:43.00, network_id:2, shared_bundle:10 }, { network:"TELECEL", size:"15GB", price:60.00, network_id:2, shared_bundle:15 }, { network:"TELECEL", size:"20GB", price:78.00, network_id:2, shared_bundle:20 }, { network:"TELECEL", size:"30GB", price:115.00, network_id:2, shared_bundle:30 }, { network:"AirtelTigo", size:"1GB", price:4.70, network_id:1, shared_bundle:1 }, { network:"AirtelTigo", size:"2GB", price:9.30, network_id:1, shared_bundle:2 }, { network:"AirtelTigo", size:"3GB", price:14.00, network_id:1, shared_bundle:3 }, { network:"AirtelTigo", size:"4GB", price:18.50, network_id:1, shared_bundle:4 }, { network:"AirtelTigo", size:"5GB", price:23.00, network_id:1, shared_bundle:5 }, { network:"AirtelTigo", size:"6GB", price:27.50, network_id:1, shared_bundle:6 }, { network:"AirtelTigo", size:"7GB", price:32.00, network_id:1, shared_bundle:7 }, { network:"AirtelTigo", size:"8GB", price:35.50, network_id:1, shared_bundle:8 }, { network:"AirtelTigo", size:"9GB", price:39.00, network_id:1, shared_bundle:9 }, { network:"AirtelTigo", size:"10GB", price:43.00, network_id:1, shared_bundle:10 }, { network:"AirtelTigo", size:"15GB", price:60.00, network_id:4, shared_bundle:15 }, { network:"AirtelTigo", size:"20GB", price:73.00, network_id:4, shared_bundle:20 }, { network:"AirtelTigo", size:"30GB", price:83.00, network_id:4, shared_bundle:30 }, { network:"AirtelTigo", size:"40GB", price:93.00, network_id:4, shared_bundle:40 }, { network:"AirtelTigo", size:"50GB", price:119.00, network_id:4, shared_bundle:50 }, { network:"AirtelTigo", size:"60GB", price:150.00, network_id:4, shared_bundle:60 }, { network:"AirtelTigo", size:"80GB", price:187.00, network_id:4, shared_bundle:80 }, { network:"AirtelTigo", size:"100GB", price:204.00, network_id:4, shared_bundle:100 }, { network:"AirtelTigo", size:"200GB", price:390.00, network_id:4, shared_bundle:200 } ];
        const disabledPackages = <?php echo json_encode( get_option( 'highest_disabled_packages', array() ) ); ?>;
        let currentBuyPackage = null;
        function renderWalkin() { let html = ''; bundles.forEach(pkg => { if (disabledPackages.includes(pkg.network + '-' + pkg.size)) return; const colorClass = pkg.network === 'MTN' ? 'text-[#FFD700]' : (pkg.network === 'TELECEL' ? 'text-red-500' : 'text-blue-500'); const btnClass = pkg.network === 'MTN' ? 'bg-[#FFD700] text-[#001F3F] hover:bg-yellow-400' : (pkg.network === 'TELECEL' ? 'bg-red-600 text-white hover:bg-red-700' : 'bg-blue-600 text-white hover:bg-blue-700'); html += `<div onclick="startBuy('${pkg.network}','${pkg.size}',${pkg.price},${pkg.network_id},${pkg.shared_bundle})" class="bg-white border-2 border-slate-200 hover:border-[#001F3F] p-4 md:p-5 rounded-3xl flex flex-col min-h-[170px] transition-all hover:shadow-2xl cursor-pointer"><div class="flex flex-col"><p class="text-xs uppercase font-bold ${colorClass}">${pkg.network}</p><p class="text-2xl leading-none font-bold text-slate-800 mt-1">${pkg.size}</p><div class="mt-4 text-lg font-bold text-emerald-600">GH₵${pkg.price.toFixed(2)}</div></div><button class="mt-auto w-full py-3 ${btnClass} font-bold rounded-3xl transition-all text-sm">BUY NOW</button></div>`; }); document.getElementById('walkin-grid').innerHTML = html; }
        function startBuy(network, size, price, network_id, shared_bundle) { <?php if ( $disable_public_buy || $disable_all_buy ) : ?>alert("Buying is currently disabled by admin."); return;<?php endif; ?> if (!PAYSTACK_PUBLIC_KEY) { alert("Paystack not configured."); return; } currentBuyPackage = { network, size, price, network_id, shared_bundle }; showBuyModal(); }
        function showBuyModal() { const m = document.getElementById('buy-phone-modal'); m.classList.remove('hidden'); m.classList.add('flex'); const pi = document.getElementById('modal-phone'); pi.value = document.getElementById('walkin-phone').value.trim() || ''; pi.focus(); }
        function closeBuyModal() { const m = document.getElementById('buy-phone-modal'); m.classList.add('hidden'); m.classList.remove('flex'); }
        function submitBuyPhone() { const phone = document.getElementById('modal-phone').value.trim(); if (!phone) { alert("⚠️ Please enter the customer's telephone number"); return; } const emailInput = document.getElementById('modal-email').value.trim(); const email = emailInput ? emailInput : phone + '@highestdataplug.com'; document.getElementById('walkin-phone').value = phone; closeBuyModal(); const pkg = currentBuyPackage; if (!pkg) return; const paystackRef = 'HDP-' + Date.now() + '-' + Math.floor(Math.random() * 1000000); const logData = new FormData(); logData.append('action', 'highest_log_payment_initiation'); logData.append('nonce', HIGHEST_NONCE); logData.append('ref', paystackRef); logData.append('amount', pkg.price); logData.append('phone', phone); logData.append('email', email); logData.append('package_name', pkg.network + '-' + pkg.size); logData.append('network_id', pkg.network_id); logData.append('shared_bundle', pkg.shared_bundle); logData.append('selling_price', pkg.price); logData.append('order_type', 'walkin'); fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: logData }).then(r => r.json()).then(res => console.log('Payment session logged:', res)).catch(e => console.error('Payment session log error:', e)); var gateway = "<?php echo esc_js( get_option('highest_payment_gateway','paystack') ); ?>"; if (gateway === 'moolre') { var mfd = new FormData(); mfd.append('action', 'highest_get_moolre_checkout_url'); mfd.append('nonce', HIGHEST_NONCE); mfd.append('ref', paystackRef); mfd.append('amount', pkg.price); mfd.append('phone', phone); mfd.append('email', email); mfd.append('description', pkg.network + '-' + pkg.size + ' to ' + phone); fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: mfd }).then(r => r.json()).then(res => { if (res.success) { window.location.href = res.data.url; } else { alert("❌ Payment error: " + res.data); } }); return; } const handler = PaystackPop.setup({ key: PAYSTACK_PUBLIC_KEY, email: email, amount: Math.round(pkg.price * 100), currency: "GHS", ref: paystackRef, callback: function(response) { const data = new FormData(); data.append('action', 'highest_buy_data_verified'); data.append('reference', response.reference); data.append('trans', response.trans); data.append('nonce', HIGHEST_NONCE); data.append('phone', phone); data.append('email', email); data.append('package_name', pkg.network + '-' + pkg.size); data.append('network_id', pkg.network_id); data.append('shared_bundle', pkg.shared_bundle); data.append('selling_price', pkg.price); data.append('agent_id', '0'); data.append('order_type', 'walkin'); fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: data }).then(r => r.json()).then(res => { if (res.success) { var wMsg = <?php echo json_encode( $this->get_delivery_message( 'highest_walkin_success_msg', '✅ Order placed successfully! Delivery usually takes about {minutes} minute(s).' ) ); ?>; alert(wMsg); document.getElementById('walkin-phone').value = ''; } else { alert("❌ " + (res.data || "Something went wrong.")); } }).catch(() => { fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: data }).then(r => r.json()).then(res => { if (res.success) { var wMsg = <?php echo json_encode( $this->get_delivery_message( 'highest_walkin_success_msg', '✅ Order placed successfully! Delivery usually takes about {minutes} minute(s).' ) ); ?>; alert(wMsg); document.getElementById('walkin-phone').value = ''; } else { alert("❌ " + (res.data || "Something went wrong.")); } }).catch(() => alert("❌ Network error. Please try again.")); }); }, onClose: function() {} }); handler.openIframe(); }
        function showRegisterModal() { const m = document.getElementById('register-modal'); m.classList.remove('hidden'); m.classList.add('flex'); }
        function closeRegisterModal() { const m = document.getElementById('register-modal'); m.classList.add('hidden'); m.classList.remove('flex'); }
        function toggleAgentMenu() { document.getElementById('agent-menu-dropdown').classList.toggle('hidden'); }
        // autoCheckProcessingInResult removed
        function publicTrackOrder() { const phone = document.getElementById('public-tracker-phone').value.trim(); const resultDiv = document.getElementById('public-tracker-result'); if (!phone) { resultDiv.innerHTML = '<p class="text-red-500">Please enter your phone number</p>'; return; } resultDiv.innerHTML = '<p class="text-center py-8">Searching orders &amp; checking live status...</p>'; const fd = new FormData(); fd.append('action', 'highest_public_track_order'); fd.append('nonce', HIGHEST_NONCE); fd.append('phone', phone); fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: fd }).then(r => r.json()).then(res => { if (res.success && res.data && res.data.length) { let html = '<h3 class="font-semibold mb-4 text-emerald-700">Your Orders</h3><table class="w-full text-sm"><thead><tr class="border-b"><th class="text-left py-3">Date</th><th class="text-left py-3">Package</th><th class="text-center py-3">Status</th></tr></thead><tbody>'; res.data.forEach(order => { const dateStr = order.created_at ? new Date(order.created_at).toLocaleDateString('en-GB') : '—'; const st = (order.status || 'PROCESSING').toUpperCase(); const sc = st === 'DELIVERED' ? 'bg-emerald-100 text-emerald-700' : (st === 'FAILED' ? 'bg-red-100 text-red-700' : 'bg-amber-100 text-amber-700'); html += `<tr class="border-b"><td class="py-3">${dateStr}</td><td class="py-3 font-semibold">${order.package}</td><td class="py-3 text-center"><span id="status-${order.ref}" class="px-3 py-1 rounded-3xl text-xs ${sc}">${st}</span></td></tr>`; }); html += '</tbody></table>'; resultDiv.innerHTML = html; } else { resultDiv.innerHTML = '<p class="text-gray-500">No orders found for this number yet.</p>'; } }).catch(() => { resultDiv.innerHTML = '<p class="text-red-500">Error checking status.</p>'; }); }
        // checkLiveStatus removed
        document.addEventListener('DOMContentLoaded', function() { const form = document.getElementById('register-form'); const btn = document.getElementById('submit-btn'); form.addEventListener('submit', function(e) { e.preventDefault(); btn.disabled = true; btn.innerHTML = '<i class="fas fa-spinner fa-spin mr-2"></i> Creating...'; const params = `action=highest_register_agent&name=${encodeURIComponent(document.getElementById('reg-name').value.trim())}&phone=${encodeURIComponent(document.getElementById('reg-phone').value.trim())}&email=${encodeURIComponent(document.getElementById('reg-email').value.trim())}&password=${encodeURIComponent(document.getElementById('reg-password').value.trim())}&ref=${encodeURIComponent(new URLSearchParams(window.location.search).get('ref')||'')}&nonce=<?php echo $nonce; ?>`; fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, body: params }).then(r => r.json()).then(data => { if (data.success) { alert(data.data); closeRegisterModal(); form.reset(); setTimeout(() => { window.location.href = AGENT_PORTAL_URL; }, 1500); } else { alert("❌ " + (data.data || "Registration failed")); } }).finally(() => { btn.disabled = false; btn.textContent = "Create My Agent Account"; }); }); document.querySelectorAll('input[type="tel"]').forEach(input => { input.addEventListener('input', function() { this.value = this.value.replace(/\D/g, '').slice(0, 10); }); }); renderWalkin(); });
        // ─── PWA Install for Public Customers (always visible button) ──
let deferredPromptPublic;

window.addEventListener('beforeinstallprompt', (e) => {
    // Prevent the mini-infobar
    e.preventDefault();
    // Save the event for later use
    deferredPromptPublic = e;
    // Button is already visible – just log that install is ready
    console.log('Install prompt available. Button is ready.');
});

function installPublic() {
    if (deferredPromptPublic) {
        // Native prompt available – show it
        deferredPromptPublic.prompt();
        deferredPromptPublic.userChoice.then((choiceResult) => {
            if (choiceResult.outcome === 'accepted') {
                alert('✅ App installed! You can now buy data from your home screen.');
            }
            deferredPromptPublic = null;
        });
    } else {
        // Fallback: manual instructions
        const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream;
        if (isIOS) {
            alert('📱 To install this app:\n\n1. Tap the Share button (square with arrow)\n2. Scroll down and tap "Add to Home Screen"\n3. Tap "Add"');
        } else {
            alert('📲 To install this app:\n\nOpen your browser menu (⋮) and select "Add to Home Screen" or "Install app".');
        }
    }
}

// If already running as a PWA, hide the install button
if (window.matchMedia('(display-mode: standalone)').matches) {
    document.getElementById('pwa-install-btn-public').style.display = 'none';
}
</script>
        <?php
        return ob_get_clean();
    }

    // ─── Agent Portal (v5.1 – no SOS header button; chart added; SOS prompt behind admin toggle) ──
    public function render_agent_portal() {
        if ( ! is_user_logged_in() || get_user_meta( get_current_user_id(), 'is_highest_agent', true ) !== 'yes' ) {
            ob_start(); ?>
            <div class="max-w-md mx-auto text-center p-8 bg-white rounded-[2.5rem] shadow-xl border border-slate-100 mt-10 mb-10">
                <i class="fas fa-user-shield text-5xl text-[#001F3F] mb-4"></i>
                <h2 class="text-3xl font-bold text-slate-800">Agent Portal</h2>
                <p class="text-slate-600 mt-2 mb-6">
                    <?php if ( is_user_logged_in() ) : ?>
                        You do not have access to the Agent Portal.<br>Please contact admin if you believe this is a mistake.
                    <?php else : ?>
                        You need to log in to access your agent dashboard.
                    <?php endif; ?>
                </p>
                <div class="flex items-center justify-center gap-4 flex-wrap">
                    <?php if ( ! is_user_logged_in() ) : ?>
                        <a href="<?php echo esc_url( home_url( '/agent-login/' ) ); ?>"
                           class="inline-flex items-center gap-2 px-6 py-3 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-bold rounded-2xl shadow-md transition">
                            <i class="fas fa-sign-in-alt"></i> Agent Login
                        </a>
                    <?php endif; ?>
                    <a href="<?php echo esc_url( home_url( '/' ) ); ?>"
                       class="inline-flex items-center gap-2 px-6 py-3 bg-white border-2 border-[#FFD700] text-[#001F3F] font-semibold rounded-2xl transition hover:bg-[#FFD700] hover:text-[#001F3F]">
                        <i class="fas fa-home"></i> Return to Homepage
                    </a>
                </div>
            </div>
            <?php
            return ob_get_clean();
        }
        $user_id = get_current_user_id();
        // Suspended agent flag
        $agent_suspended = get_user_meta( $user_id, 'highest_agent_suspended', true ) === '1';
        // Block unapproved agents from accessing the portal
        if ( get_user_meta( $user_id, 'highest_agent_approved', true ) === 'no' ) {
            ob_start(); ?>
            <div class="max-w-md mx-auto text-center p-8 bg-white rounded-[2.5rem] shadow-xl border border-slate-100 mt-10 mb-10">
                <i class="fas fa-clock text-5xl text-amber-500 mb-4"></i>
                <h2 class="text-3xl font-bold text-slate-800">Account Pending Approval</h2>
                <p class="text-slate-600 mt-2 mb-6">Your agent account is being reviewed. You will receive an SMS once approved.</p>
                <a href="<?php echo esc_url( home_url( '/' ) ); ?>" class="inline-block bg-[#FFD700] text-[#001F3F] px-8 py-3 rounded-2xl font-bold shadow-md">Return Home</a>
            </div>
            <?php
            return ob_get_clean();
        }
        $wallet = (float) get_user_meta( $user_id, 'highest_wallet', true );
        $profit = (float) get_user_meta( $user_id, 'highest_profit', true );
        $agent_prices = get_user_meta( $user_id, 'highest_agent_prices', true ) ?: array();

        $total_withdrawn = 0;
        $pending_withdrawals = 0;
        foreach ( ( get_user_meta( $user_id, 'highest_withdraw_requests', true ) ?: array() ) as $w ) {
            if ( isset( $w['status'] ) && $w['status'] === 'approved' ) {
                $total_withdrawn += (float) ( $w['amount'] ?? 0 );
            }
            if ( isset( $w['status'] ) && $w['status'] === 'pending_approval' ) {
                $pending_withdrawals++;
            }
        }
        $stats = array(
            array( 'label' => 'Wallet Balance', 'value' => 'GH₵' . number_format( $wallet, 2 ) ),
            array( 'label' => 'Profit Balance', 'value' => 'GH₵' . number_format( $profit, 2 ) ),
            array( 'label' => 'Lifetime Withdrawals', 'value' => 'GH₵' . number_format( $total_withdrawn, 2 ) ),
            array( 'label' => 'Pending Withdrawals', 'value' => (string) $pending_withdrawals ),
        );

        global $wpdb;
        $table_orders = $wpdb->prefix . 'highest_orders';
        $total_sales = $wpdb->get_var( $wpdb->prepare( "SELECT SUM(selling_price) FROM $table_orders WHERE user_id = %d AND status = 'DELIVERED'", $user_id ) ) ?: 0;
        $badge = 'Bronze';
        $badge_color = 'text-amber-700 bg-amber-100';
        $next_target = 500;
        if ( $total_sales >= 5000 ) {
            $badge = 'Diamond 💎';
            $badge_color = 'text-cyan-600 bg-cyan-100';
            $next_target = 5000;
        } elseif ( $total_sales >= 2000 ) {
            $badge = 'Gold 🏆';
            $badge_color = 'text-yellow-700 bg-yellow-100';
            $next_target = 5000;
        } elseif ( $total_sales >= 500 ) {
            $badge = 'Silver 🥈';
            $badge_color = 'text-slate-600 bg-slate-200';
            $next_target = 2000;
        }
        $progress_percent = $total_sales >= 5000 ? 100 : min( 100, ( $total_sales / $next_target ) * 100 );

        $default_bundles = $this->get_default_agent_prices();

        // Pre-fetch wholesale prices once (avoid one DB query per package in the loop).
        // get_subagent_wholesale_price() queries the parent on every call, so we replicate
        // the logic here with a single query.
        $portal_wholesale_prices = array();
        if ( get_option( 'highest_enable_wholesale_pricing', 'no' ) === 'yes' ) {
            global $wpdb;
            // Only load wholesale prices when allow_wholesale = 1 for this sub-agent.
            // COALESCE( allow_wholesale, 1 ) treats legacy NULL rows as enabled (preserves prior behaviour).
            $portal_parent_id = $wpdb->get_var( $wpdb->prepare(
                "SELECT parent_agent_id FROM {$wpdb->prefix}highest_agent_referrals
                 WHERE child_agent_id = %d
                   AND ( status = 'active' OR status IS NULL OR status = '' )
                   AND COALESCE( allow_wholesale, 1 ) = 1
                 LIMIT 1",
                $user_id
            ) );
            if ( $portal_parent_id ) {
                $portal_wholesale_prices = get_user_meta( $portal_parent_id, 'highest_wholesale_prices', true ) ?: array();
            }
        }

        $agent_bundles = array();
        foreach ( $default_bundles as $key => $default_price ) {
            // Determine the agent's actual buy cost: wholesale (if set) → tier → default.
            if ( isset( $portal_wholesale_prices[ $key ] ) && $portal_wholesale_prices[ $key ] > 0 ) {
                $agent_cost = (float) $portal_wholesale_prices[ $key ];
            } else {
                $agent_cost = $this->get_agent_cost( $user_id, $key );
                if ( $agent_cost <= 0 ) $agent_cost = $default_price;
            }

            $current = $agent_prices[ $key ] ?? $agent_cost;
            if ( $current < $agent_cost ) {
                $current = $agent_cost;
            }
            $key_parts = explode( '-', $key );
            $network_raw = $key_parts[0];
            $size = $key_parts[ count( $key_parts ) - 1 ];
            $network = $network_raw === 'Telecel' ? 'TELECEL' : $network_raw;
            $network_id = 3;
            if ( stripos( $key, 'Telecel' ) !== false ) {
                $network_id = 2;
            } elseif ( stripos( $key, 'ATiShare' ) !== false ) {
                $network_id = 1; // Fix: ATiShare = network 1, not 4
            } elseif ( stripos( $key, 'ATBigTime' ) !== false ) {
                $network_id = 4;
            }
            $agent_bundles[] = array(
                'key'           => $key,
                'network'       => $network,
                'size'          => $size,
                'price'         => $current,
                'min_price'     => $agent_cost, // Fix: actual cost, not hardcoded default
                'network_id'    => $network_id,
                'shared_bundle' => (int) filter_var( $size, FILTER_SANITIZE_NUMBER_INT ),
            );
        }

        $announcements = get_option( 'highest_announcements', array() );
        $disable_agent_buy = get_option( 'highest_disable_agent_buy', false );
        $disable_all_buy = get_option( 'highest_disable_all_buy', false );
        $whatsapp = get_option( 'highest_whatsapp_number', '0551907463' );
        $login_streak = (int) get_user_meta( $user_id, 'highest_login_streak', true );

        ob_start();
        ?>
        <div class="min-h-screen bg-slate-100 flex font-sans">
            <div class="lg:hidden fixed top-4 left-4 z-50">
                <button onclick="toggleMobileMenu()" class="bg-[#001F3F] text-white p-3 rounded-2xl shadow-lg"><i class="fas fa-bars text-xl"></i></button>
            </div>
            <aside id="agent-sidebar" class="w-64 bg-[#001F3F] text-white shadow-2xl py-4 px-3 fixed lg:static inset-y-0 left-0 transform -translate-x-full lg:translate-x-0 transition-transform duration-300 z-40 lg:z-auto flex flex-col">
    <div class="flex justify-end mb-4">
        <a href="<?php echo wp_logout_url( home_url() ); ?>" class="flex items-center gap-1 px-3 py-1.5 bg-white/10 hover:bg-red-500/20 text-red-300 rounded-xl text-xs font-medium transition">
            <i class="fas fa-right-from-bracket text-xs"></i> Logout
        </a>
    </div>
    <div class="flex items-center gap-2 mb-6">
        <img src="https://highestdataplug.com/wp-content/uploads/2026/04/20ybve.jpg" class="h-8 w-auto rounded-full">
        <h1 class="text-lg font-bold text-[#FFD700] leading-tight">Agent Portal</h1>
    </div>
    <nav class="flex-1 space-y-0.5 overflow-y-auto" id="agent-nav">
        <!-- Group: Main -->
        <div class="text-[10px] uppercase text-slate-400 px-2 mt-3 mb-1 font-semibold tracking-wider">Main</div>
        <a onclick="switchSection('dashboard');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm active">
            <i class="fas fa-house w-4 text-center"></i><span>Dashboard</span>
        </a>
        <a onclick="switchSection('buy');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-bolt w-4 text-center"></i><span>Buy Data</span>
        </a>
        <!-- Group: Finance -->
        <div class="text-[10px] uppercase text-slate-400 px-2 mt-3 mb-1 font-semibold tracking-wider">Finance</div>
        <a onclick="switchSection('wallet');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-wallet w-4 text-center"></i><span>Wallet Top-up</span>
        </a>
        <a onclick="switchSection('topup-history');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-history w-4 text-center text-blue-400"></i><span>Wallet History</span>
        </a>
        <a onclick="switchSection('withdraw');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-hand-holding-dollar w-4 text-center"></i><span>Withdraw Profit</span>
        </a>
        <!-- Group: Store & Prices -->
        <div class="text-[10px] uppercase text-slate-400 px-2 mt-3 mb-1 font-semibold tracking-wider">Store</div>
        <a onclick="switchSection('prices');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-tags w-4 text-center text-purple-400"></i><span>My Prices</span>
        </a>
        <a onclick="switchSection('storefront');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-store w-4 text-center"></i><span>My Storefront</span>
        </a>
        <a onclick="switchSection('afa');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-id-card w-4 text-center text-purple-400"></i><span>AFA Registration</span>
        </a>
        <?php if ( get_option( 'highest_enable_referral_system', 'no' ) === 'yes' && $this->get_parent_permission( $user_id, 'allow_referral' ) ) : ?>
        <a onclick="switchSection('referrals');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-link w-4 text-center text-indigo-400"></i><span>My Referrals</span>
        </a>
        <?php endif; ?>
        <!-- Group: Activity -->
        <div class="text-[10px] uppercase text-slate-400 px-2 mt-3 mb-1 font-semibold tracking-wider">Activity</div>
        <a onclick="switchSection('transactions');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-list-ul w-4 text-center"></i><span>Transactions</span>
        </a>
        <a onclick="switchSection('store-attempts');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-eye w-4 text-center text-cyan-400"></i><span>Store Attempts</span>
        </a>
        <a onclick="switchSection('payment-history');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-clock-rotate-left w-4 text-center text-indigo-400"></i><span>Payment History</span>
        </a>
        <!-- Group: Account -->
        <div class="text-[10px] uppercase text-slate-400 px-2 mt-3 mb-1 font-semibold tracking-wider">Account</div>
        <a onclick="switchSection('profile');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-user-circle w-4 text-center"></i><span>Profile</span>
        </a>
        <a onclick="switchSection('waec-tracker');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-graduation-cap w-4 text-center text-purple-400"></i><span>WAEC Tracker</span>
        </a>
        <?php if ( get_option( 'highest_enable_tier_upgrade', 'no' ) === 'yes' && $this->get_parent_permission( $user_id, 'allow_upgrade' ) && ! $this->subagent_has_wholesale_enabled( $user_id ) ) : ?>
        <a onclick="switchSection('tier-upgrade');toggleMobileMenu()" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-arrow-up w-4 text-center text-purple-400"></i><span>Upgrade Tier</span>
        </a>
        <?php endif; ?>
        <a href="https://test.highestdataplug.com" target="_blank" class="nav-item flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-white/10 cursor-pointer text-sm">
            <i class="fas fa-store w-4 text-center text-yellow-400"></i><span>Marketplace</span>
        </a>
    </nav>
    <a href="<?php echo wp_logout_url( home_url() ); ?>" class="flex items-center gap-2 px-3 py-2 rounded-xl hover:bg-red-500/20 text-red-400 cursor-pointer transition-colors text-sm font-medium mt-auto">
        <i class="fas fa-right-from-bracket w-4 text-center"></i> Logout
    </a>
</aside>
            <main class="flex-1 p-6 md:p-10 overflow-y-auto pb-24 lg:pb-10">
                <div class="mb-8 flex justify-end">
                    <button onclick="switchSection('transactions')" class="flex items-center gap-3 bg-[#001F3F] hover:bg-[#001F3F]/90 text-[#FFD700] px-6 py-3 rounded-3xl font-semibold text-base shadow-lg transition-all"><i class="fas fa-search"></i><span>Track Orders (Live)</span></button>
                </div>
                <?php if ( ! empty( $announcements ) ) : ?>
                <div class="mb-8 bg-amber-100 border border-amber-300 p-5 rounded-3xl text-slate-800">
                    <h4 class="font-bold flex items-center gap-2"><i class="fas fa-bullhorn"></i> Announcements</h4>
                    <?php foreach ( array_reverse( $announcements ) as $ann ) : ?>
                        <div class="mt-3"><strong><?php echo esc_html( $ann['title'] ); ?></strong><br><?php echo esc_html( $ann['message'] ); ?></div>
                    <?php endforeach; ?>
                </div>
                <?php endif; ?>
                <div id="section-dashboard" class="section">
                    <header class="flex flex-col md:flex-row md:items-center justify-between gap-4 mb-10">
                        <div>
                            <h2 class="text-3xl font-bold text-slate-800 flex items-center gap-3">
                                Welcome back, <?php echo esc_html( wp_get_current_user()->display_name ); ?>
                                <span class="text-sm px-3 py-1 rounded-full font-bold <?php echo $badge_color; ?>"><?php echo $badge; ?> Agent</span>
                            </h2>
                            <p class="text-slate-500 mt-1">Manage your data business and track profits effortlessly.</p>
                            <?php
                            $current_tier_slug = get_user_meta( $user_id, 'highest_agent_tier', true ) ?: 'standard';
                            $all_tiers = get_option( 'highest_agent_tiers', array( 'standard' => 'Standard' ) );
                            ?>
                            <p class="text-sm text-slate-400 mt-1">Your Tier: <strong><?php echo esc_html( $all_tiers[ $current_tier_slug ] ?? 'Standard' ); ?></strong></p>
                    <?php if ( $agent_suspended ) : ?>
                    <div class="mt-3 bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded-2xl text-sm font-medium"><i class="fas fa-ban mr-2"></i> Your account is currently <strong>suspended</strong>. Purchases and withdrawals are disabled. Contact admin.</div>
                    <?php endif; ?>
                            <?php if ( $total_sales < 5000 ) : ?>
                            <div class="mt-4 max-w-sm"><div class="flex justify-between text-xs text-slate-500 mb-1"><span>GH₵<?php echo number_format( $total_sales, 2 ); ?></span><span>Goal: GH₵<?php echo number_format( $next_target, 2 ); ?></span></div><div class="w-full bg-slate-200 rounded-full h-2.5"><div class="bg-emerald-500 h-2.5 rounded-full" style="width: <?php echo $progress_percent; ?>%"></div></div></div>
                            <?php endif; ?>
                            <p class="text-xs text-slate-400 mt-2">🔥 Login Streak: <strong><?php echo $login_streak; ?> day(s)</strong></p>
                        </div>
                        <button onclick="topUpWallet()" class="bg-[#001F3F] text-[#FFD700] px-8 py-3 rounded-2xl font-bold shadow-lg hover:scale-105 transition-transform">
    <i class="fas fa-plus mr-2"></i> Fund Wallet
</button>
<?php if ( get_option( 'highest_enable_tier_upgrade', 'no' ) === 'yes' && $this->get_parent_permission( $user_id, 'allow_upgrade' ) && ! $this->subagent_has_wholesale_enabled( $user_id ) ) : ?>
<button onclick="switchSection('tier-upgrade')" class="bg-purple-600 text-white px-6 py-3 rounded-2xl font-bold shadow-lg hover:scale-105 transition-transform">
    <i class="fas fa-arrow-up mr-2"></i> Upgrade Tier
</button>
<?php endif; ?>
<button id="pwa-install-btn" class="bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] px-6 py-3 rounded-2xl font-bold shadow-lg transition-transform" onclick="installApp()">
    <i class="fas fa-download mr-2"></i> Install App
</button>
</header>
                    <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-6 mb-10">
                        <?php foreach ( $stats as $stat ) : ?>
                            <div class="bg-white p-6 rounded-[2rem] shadow-sm border border-slate-200"><p class="text-slate-500 text-sm font-medium mb-1 uppercase tracking-wider"><?php echo $stat['label']; ?></p><p class="text-3xl font-bold text-slate-800"><?php echo $stat['value']; ?></p></div>
                        <?php endforeach; ?>
                    </div>
                    <!-- v5.3: Quick-action shortcut buttons -->
                    <div class="flex flex-wrap gap-3 mb-8">
                        <button onclick="switchSection('buy')" class="bg-[#001F3F] text-white px-4 py-2 rounded-xl text-sm font-medium hover:opacity-90 transition"><i class="fas fa-bolt mr-1"></i> Buy Data</button>
                        <button onclick="switchSection('prices')" class="bg-purple-600 text-white px-4 py-2 rounded-xl text-sm font-medium hover:opacity-90 transition"><i class="fas fa-tags mr-1"></i> My Prices</button>
                        <button onclick="switchSection('storefront')" class="bg-teal-600 text-white px-4 py-2 rounded-xl text-sm font-medium hover:opacity-90 transition"><i class="fas fa-store mr-1"></i> My Storefront</button>
                        <?php if ( get_option( 'highest_enable_referral_system', 'no' ) === 'yes' && $this->get_parent_permission( $user_id, 'allow_referral' ) ) : ?>
                        <button onclick="switchSection('referrals')" class="bg-indigo-600 text-white px-4 py-2 rounded-xl text-sm font-medium hover:opacity-90 transition"><i class="fas fa-users mr-1"></i> Referrals</button>
                        <?php endif; ?>
                        <button onclick="switchSection('withdraw')" class="bg-amber-500 text-white px-4 py-2 rounded-xl text-sm font-medium hover:opacity-90 transition"><i class="fas fa-hand-holding-dollar mr-1"></i> Withdraw</button>
                    </div>
                    <!-- 7-day sales chart -->
                    <div class="bg-white p-6 rounded-[2rem] shadow-sm border border-slate-200 mb-10">
                        <h3 class="text-lg font-semibold mb-4">📈 7‑Day Sales</h3>
                        <canvas id="agent-sales-chart" height="200"></canvas>
                    </div>
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-10">
                        <div class="bg-white p-6 rounded-[2rem] shadow-sm border border-slate-200">
                            <h3 class="text-lg font-semibold mb-4">📊 This Month</h3>
                            <div id="monthly-stats-container" class="space-y-2 text-sm text-slate-600"><p class="text-slate-400 italic">Loading...</p></div>
                        </div>
                        <div class="bg-white p-6 rounded-[2rem] shadow-sm border border-slate-200">
                            <h3 class="text-lg font-semibold mb-4">🏷️ Best Selling Packages</h3>
                            <div id="top-packages-container" class="text-sm"><p class="text-slate-400 italic">Loading...</p></div>
                        </div>
                    </div>
                    <section class="bg-[#001F3F] text-white p-8 rounded-[2.5rem] shadow-xl"><h3 class="text-xl font-bold mb-6">Recent Activity</h3><div id="orders-container-mini" class="space-y-4"></div></section>
                </div>
                <div id="section-wallet" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2"><i class="fas fa-wallet text-emerald-500"></i> Auto Wallet Top-up via Paystack</h4>
                        <div class="flex flex-col sm:flex-row gap-4"><input type="number" id="topup-amount" placeholder="Enter amount (GH₵)" step="0.01" min="1" class="flex-1 p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none"><button onclick="topUpWallet()" class="bg-[#001F3F] text-[#FFD700] px-10 py-5 rounded-3xl font-bold hover:scale-105 transition-transform flex items-center justify-center gap-2"><i class="fas fa-credit-card"></i> Pay via Paystack</button></div>
                        <p class="text-xs text-slate-500 mt-3">Wallet is funded instantly upon successful payment verification.</p>
                        <div class="mt-4 bg-amber-50 border border-amber-200 p-4 rounded-2xl"><p class="text-sm font-medium text-amber-800 mb-2"><i class="fas fa-exclamation-triangle"></i> Top-up not credited? Enter your payment reference below and retry.</p><div class="flex flex-col sm:flex-row gap-2"><input type="text" id="retry-topup-ref" placeholder="Paystack reference" class="flex-1 p-3 bg-white border border-slate-200 rounded-xl text-sm"><button onclick="retryTopUp()" class="px-6 py-3 bg-amber-500 hover:bg-amber-600 text-white rounded-xl font-semibold text-sm">Retry Verification</button></div></div>
                    </div>
                    <div class="mt-8 bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2"><i class="fas fa-hand-holding-dollar text-amber-500"></i> Manual Top-up (Mobile Money)</h4>
                        <p class="text-slate-600 mb-2">Send Mobile Money to <strong class="text-[#001F3F]">0551907463</strong></p><p class="text-xs text-slate-500 mb-6">After sending, fill the form below to request credit.</p>
                        <div class="space-y-4"><input type="number" id="manual-topup-amount" placeholder="Amount you sent (GH₵)" step="0.01" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none"><button onclick="requestManualTopup()" class="w-full py-5 bg-amber-500 hover:bg-amber-600 text-white font-bold text-base rounded-3xl transition-all flex items-center justify-center gap-2"><i class="fas fa-paper-plane"></i> REQUEST MANUAL CREDIT</button></div>
                    </div>
                </div>
                <div id="section-topup-history" class="section hidden"><div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200"><h4 class="text-xl font-semibold mb-6 flex items-center gap-2"><i class="fas fa-history text-blue-500"></i> My Wallet Top-up History</h4>
                <?php
                $table_topup = $wpdb->prefix . 'highest_topup_history';
                $my_topups = $wpdb->get_results( $wpdb->prepare( "SELECT * FROM $table_topup WHERE user_id = %d ORDER BY created_at DESC LIMIT 50", $user_id ) );
                if ( empty( $my_topups ) ) { echo '<p class="text-slate-500 italic">No wallet funding history found.</p>'; }
                else { echo '<ul class="space-y-3">';
                    foreach ( $my_topups as $tp ) {
                        $date_formatted = date( 'd M Y • H:i', strtotime( $tp->created_at ) );
                        $tp_amount = (float) $tp->amount;
                        $tp_sign = $tp_amount >= 0 ? '+' : '';
                        $tp_color = $tp_amount >= 0 ? 'text-emerald-600' : 'text-red-600';
                        $tp_label = strtoupper( str_replace( '_', ' ', $tp->type ) );
                        $tp_balance = isset( $tp->balance_after ) && $tp->balance_after > 0 ? 'GH₵' . number_format( (float) $tp->balance_after, 2 ) : '—';
                        echo '<li class="flex flex-col md:flex-row justify-between items-start md:items-center bg-slate-50 p-5 rounded-3xl text-sm gap-4"><div><span class="font-semibold text-base text-slate-800">' . esc_html( $tp_label ) . '</span><div class="text-xs text-slate-400 mt-1">Ref: <code>' . esc_html( $tp->reference ) . '</code></div><div class="text-xs text-slate-400 mt-0.5">Balance after: <strong>' . $tp_balance . '</strong></div></div><div class="text-right"><div class="text-xl font-bold ' . $tp_color . '">' . $tp_sign . ' GH₵' . number_format( abs($tp_amount), 2 ) . '</div><span class="inline-block mt-1 px-3 py-1 text-xs rounded-full bg-emerald-100 text-emerald-700">' . esc_html( strtoupper( $tp->status ) ) . '</span></div><div class="text-xs text-slate-400 md:text-right">' . esc_html( $date_formatted ) . '</div></li>';
                    } echo '</ul>';
                } ?></div></div>
                <div id="section-buy" class="section hidden"><section class="bg-white p-6 md:p-8 rounded-[2.5rem] shadow-sm border border-slate-200"><h4 class="text-xl font-bold mb-6 flex items-center gap-2"><i class="fas fa-bolt text-yellow-500"></i> Quick Purchase</h4><div class="space-y-4"><label class="block text-sm font-medium text-slate-600 ml-2">Recipient Phone Number</label><input id="agent-buy-phone" placeholder="0551234567" maxlength="10" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none"><div class="text-xs text-amber-600 -mt-2 flex items-center gap-1"><i class="fas fa-info-circle"></i> Verify your number (10 digits only)</div><div id="agent-buy-grid" class="grid grid-cols-2 sm:grid-cols-3 gap-3 mt-8"></div></div></section></div>
                <div id="section-prices" class="section hidden"><div class="bg-white p-6 md:p-8 rounded-[2.5rem] shadow-sm border border-slate-200"><h4 class="text-lg font-semibold mb-4 flex items-center gap-2"><i class="fas fa-tags text-purple-500"></i> My Customer Prices</h4><p class="text-xs text-slate-500 mb-6">Any amount above your agent price can be saved instantly.</p><div id="price-editor-mobile" class="mt-4 grid grid-cols-1 gap-4 text-sm"></div><div id="price-editor" class="hidden lg:grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4 text-sm"></div></div></div>
                <div id="section-storefront" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2"><i class="fas fa-store text-teal-500"></i> My Storefront</h4>
                        <p class="text-xs text-slate-500 mb-6">Set your store name. Customers will buy from your personal URL and pay via Paystack.</p>
                        <div class="space-y-6">
                            <div>
                                <label class="block text-sm font-medium text-slate-600 mb-2">Store Username (e.g. Bright → /store/Bright)</label>
                                <input id="store-name-input" type="text" value="<?php echo esc_attr( get_user_meta( $user_id, 'highest_store_name', true ) ); ?>" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none">
                            </div>
                            <div>
                                <label class="block text-sm font-medium text-slate-600 mb-2">My WhatsApp Number (for customers)</label>
                                <input id="store-whatsapp-input" type="tel" value="<?php echo esc_attr( get_user_meta( $user_id, 'highest_store_whatsapp', true ) ); ?>" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none" maxlength="15">
                            </div>
                            <div>
                                <label class="block text-sm font-medium text-slate-600 mb-2">Theme Color</label>
                                <input id="store-theme-color" type="color" value="<?php echo esc_attr( get_user_meta( $user_id, 'highest_store_theme_color', true ) ?: '#FFD700' ); ?>" class="w-full p-3 bg-slate-50 border border-slate-200 rounded-3xl outline-none" style="height:55px;">
                            </div>
                            <div>
                                <label class="block text-sm font-medium text-slate-600 mb-2">Banner Image URL (optional)</label>
                                <input id="store-banner-url" type="text" value="<?php echo esc_attr( get_user_meta( $user_id, 'highest_store_banner_url', true ) ); ?>" placeholder="https://example.com/banner.jpg" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none">
                            </div>
                            <button onclick="saveStorefront()" class="w-full py-5 bg-teal-600 hover:bg-teal-700 text-white font-bold text-base rounded-3xl transition-all flex items-center justify-center gap-2">
                                <i class="fas fa-save"></i> SAVE STORE SETTINGS
                            </button>
                            <div id="storefront-url-display" class="mt-4 p-6 bg-slate-50 rounded-3xl text-sm">
                                <?php
                                $store_name = get_user_meta( $user_id, 'highest_store_name', true );
                                $store_url = $store_name ? home_url( '/store/' . sanitize_title( $store_name ) ) : 'Save a store name to generate your customer URL';
                                ?>
                                <p class="font-medium text-slate-600">Your Customer Buy URL:</p>
                                <div class="flex items-center justify-between gap-3 bg-white border border-slate-200 p-3 rounded-2xl mt-2">
                                    <a href="<?php echo esc_url( $store_url ); ?>" target="_blank" class="text-[#001F3F] break-all underline truncate" id="store-url-text"><?php echo esc_html( $store_url ); ?></a>
                                    <?php if ( $store_name ) : ?>
                                    <button onclick="navigator.clipboard.writeText(document.getElementById('store-url-text').href);alert('✅ Store link copied!');" class="bg-slate-200 hover:bg-slate-300 text-slate-700 px-4 py-2 rounded-xl text-xs font-bold shrink-0"><i class="fas fa-copy"></i> Copy</button>
                                    <?php endif; ?>
                                </div>
                                <p class="text-xs text-slate-400 mt-2">Share this link with customers – they can browse your prices and pay securely via Paystack.</p>

                                <!-- USSD Extension -->
                                <?php
                                $ussd_shortcode = rtrim( get_option( 'highest_ussd_shortcode', '*919*909*' ), '#' );
                                if ( substr( $ussd_shortcode, -1 ) !== '*' ) $ussd_shortcode .= '*';
                                $ussd_display = $ussd_shortcode . $user_id . '#';
                                ?>
                                <div class="mt-4 pt-4 border-t border-slate-200">
                                    <p class="font-medium text-slate-600">Your USSD Extension:</p>
                                    <div class="flex items-center justify-between gap-3 bg-white border border-slate-200 p-3 rounded-2xl mt-2">
                                        <code class="text-lg font-bold text-[#001F3F]" id="ussd-extension-text"><?php echo esc_html( $ussd_display ); ?></code>
                                        <button onclick="navigator.clipboard.writeText('<?php echo esc_js( $ussd_display ); ?>'); alert('✅ USSD code copied!');" class="bg-slate-200 hover:bg-slate-300 text-slate-700 px-4 py-2 rounded-xl text-xs font-bold shrink-0">
                                            <i class="fas fa-copy"></i> Copy
                                        </button>
                                    </div>
                                    <p class="text-xs text-slate-400 mt-2">Customers dial this code to buy data directly from you offline – no internet needed.</p>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- AFA REGISTRATION -->
                <div id="section-afa" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2"><i class="fas fa-id-card text-purple-500"></i> MTN AFA Registration</h4>
                        <p class="text-slate-600 mb-6">Register a customer for MTN AFA (Agent Financial Account). Cost: ₵15.00 (wallet or Paystack).</p>
                        <form id="afa-form" class="space-y-5">
                            <div><label class="block text-sm font-medium text-slate-600 mb-2">Customer Phone Number *</label>
                                <input type="tel" id="afa-phone" placeholder="0551234567" maxlength="10" class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl text-lg focus:ring-4 focus:ring-purple-500/20 outline-none" required></div>
                            <div><label class="block text-sm font-medium text-slate-600 mb-2">Full Name *</label>
                                <input type="text" id="afa-fullname" placeholder="e.g. John Doe" class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl text-lg focus:ring-4 focus:ring-purple-500/20 outline-none" required></div>
                            <div><label class="block text-sm font-medium text-slate-600 mb-2">ID Type *</label>
                                <select id="afa-idtype" class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl text-lg focus:ring-4 focus:ring-purple-500/20 outline-none" required>
                                    <option value="">-- Select --</option>
                                    <option value="Ghana Card">Ghana Card</option>
                                    <option value="Voter ID">Voter ID</option>
                                    <option value="Passport">Passport</option>
                                    <option value="Driver's License">Driver's License</option>
                                    <option value="NHIS">NHIS</option>
                                </select></div>
                            <div><label class="block text-sm font-medium text-slate-600 mb-2">ID Number *</label>
                                <input type="text" id="afa-idnumber" placeholder="e.g. GHA-123456789-0" class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl text-lg focus:ring-4 focus:ring-purple-500/20 outline-none" required></div>
                            <div><label class="block text-sm font-medium text-slate-600 mb-2">Location *</label>
                                <input type="text" id="afa-location" placeholder="e.g. Tafo, Kumasi" class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl text-lg focus:ring-4 focus:ring-purple-500/20 outline-none" required></div>
                            <div><label class="block text-sm font-medium text-slate-600 mb-2">Date of Birth *</label>
                                <input type="date" id="afa-dob" class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl text-lg focus:ring-4 focus:ring-purple-500/20 outline-none" required></div>
                            <div><label class="block text-sm font-medium text-slate-600 mb-2">Occupation *</label>
                                <input type="text" id="afa-occupation" placeholder="e.g. Teacher" class="w-full p-4 bg-slate-50 border border-slate-200 rounded-2xl text-lg focus:ring-4 focus:ring-purple-500/20 outline-none" required></div>
                            <div class="flex gap-4 pt-4">
                                <button type="button" onclick="submitAFA('wallet')" class="flex-1 py-4 bg-emerald-600 hover:bg-emerald-700 text-white font-bold rounded-2xl transition-all flex items-center justify-center gap-2"><i class="fas fa-wallet"></i> Pay with Wallet (₵15)</button>
                                <button type="button" onclick="submitAFA('paystack')" class="flex-1 py-4 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-bold rounded-2xl transition-all flex items-center justify-center gap-2"><i class="fas fa-credit-card"></i> Pay via Paystack (₵15)</button>
                            </div>
                        </form>
                    </div>
                </div>

                <div id="section-withdraw" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2"><i class="fas fa-hand-holding-dollar text-amber-500"></i> Withdraw Profit</h4>
                        <?php if ( $agent_suspended ) : ?>
                            <div class="bg-red-100 border border-red-300 text-red-700 px-4 py-3 rounded-2xl text-sm mb-4 font-medium"><i class="fas fa-ban mr-2"></i> Your account is suspended. Withdrawals are currently disabled. Please contact admin.</div>
                        <?php else : ?>
                            <?php if ( isset( $_GET['withdraw'] ) ) :
                                $wd_status = $_GET['withdraw'];
                                $wd_msgs = array(
                                    'no_momo'        => array( 'error', '⚠️ Please fill in your MoMo provider and phone number.' ),
                                    'no_account'     => array( 'error', '⚠️ Please enter your bank account number.' ),
                                    'failed_balance' => array( 'error', '⚠️ Invalid amount or insufficient profit balance.' ),
                                    'transfer_failed'=> array( 'error', '❌ MoMo transfer failed. Your profit has been refunded. Please try again.' ),
                                    'suspended'      => array( 'error', '❌ Your account is suspended. Withdrawals are disabled.' ),
                                    'success'        => array( 'success', '✅ Withdrawal request submitted. Awaiting admin approval.' ),
                                    'instant_success'=> array( 'success', '✅ MoMo withdrawal sent successfully!' ),
                                    'auto_approved'  => array( 'success', '✅ Manual withdrawal request submitted.' ),
                                );
                                if ( isset( $wd_msgs[ $wd_status ] ) ) :
                                    list( $wd_type, $wd_text ) = $wd_msgs[ $wd_status ];
                                    $wd_class = $wd_type === 'success' ? 'bg-emerald-50 border-emerald-300 text-emerald-700' : 'bg-red-100 border-red-300 text-red-700';
                            ?>
                                <div class="<?php echo $wd_class; ?> border px-4 py-3 rounded-2xl text-sm mb-4"><?php echo esc_html( $wd_text ); ?></div>
                            <?php endif; endif; ?>
                            <form method="post" class="space-y-6" id="withdraw-form">
                                <div>
                                    <label class="block text-sm font-medium text-slate-600 mb-2">Withdrawal Method</label>
                                    <select name="withdrawal_method" id="withdrawal-method" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none">
                                        <option value="auto">Automatic (MoMo – 1.98% fee)</option>
                                        <option value="manual">Manual (Bank Transfer)</option>
                                    </select>
                                </div>
                                <div>
                                    <label class="block text-sm font-medium text-slate-600 mb-2">Amount (GH₵)</label>
                                    <input type="number" name="withdraw_amount" id="withdraw-amount" placeholder="Enter amount" step="0.01" max="<?php echo esc_attr( $profit ); ?>" required class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none">
                                </div>
                                <!-- Auto (MoMo) fields -->
                                <div id="auto-fields" style="display:none;" class="space-y-4">
                                    <div>
                                        <label class="block text-sm font-medium text-slate-600 mb-2">MoMo Provider</label>
                                        <select name="momo_provider" id="momo-provider" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none">
                                            <option value="">-- Select Provider --</option>
                                            <option value="mtn">MTN Mobile Money</option>
                                            <option value="telecel">Telecel Cash</option>
                                            <option value="airteltigo">AirtelTigo Money</option>
                                        </select>
                                    </div>
                                    <div>
                                        <label class="block text-sm font-medium text-slate-600 mb-2">MoMo Phone Number</label>
                                        <input type="tel" name="momo_phone" id="momo-phone" placeholder="0551234567" maxlength="10" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none">
                                    </div>
                                    <div class="bg-amber-50 border border-amber-200 p-4 rounded-2xl text-sm space-y-1">
                                        <p>Processing fee (1.98%): <strong id="fee-display">GH₵0.00</strong></p>
                                        <p>You will receive: <strong id="net-display" class="text-emerald-600">GH₵0.00</strong></p>
                                    </div>
                                </div>
                                <!-- Manual (bank) fields -->
                                <div id="manual-fields">
                                    <label class="block text-sm font-medium text-slate-600 mb-2">Bank Account Number</label>
                                    <input type="text" name="account_number" placeholder="Enter bank account number" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none">
                                </div>
                                <button type="submit" name="hdp_withdraw_btn" class="w-full bg-amber-500 hover:bg-amber-600 text-white py-5 rounded-3xl font-bold text-lg transition-all">Request Withdrawal</button>
                            </form>
                        <?php endif; ?>
                        <!-- v5.3: Withdrawal history inline panel -->
                        <hr class="my-6 border-slate-200">
                        <button onclick="document.getElementById('withdrawal-history-panel').classList.toggle('hidden');if(!document.getElementById('withdrawal-history-panel').classList.contains('hidden')){loadWithdrawalHistory();}" class="text-indigo-600 font-medium text-sm flex items-center gap-2 hover:underline">
                            <i class="fas fa-history"></i> View Withdrawal History
                        </button>
                        <div id="withdrawal-history-panel" class="hidden mt-4">
                            <div id="withdrawal-history-container"></div>
                        </div>
                    </div>
                </div>
                <div id="section-transactions" class="section hidden"><div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200"><h4 class="text-xl font-semibold mb-6 flex items-center gap-2"><i class="fas fa-list-ul text-violet-500"></i> Transaction History • Live Status</h4><div id="txn-autocheck-notice" class="mb-4 text-xs text-amber-600 flex items-center gap-2"><i class="fas fa-spinner fa-spin"></i> Checking live delivery status for PROCESSING orders...</div><?php echo $this->render_agent_transactions( $user_id ); ?></div></div>

                <!-- STORE ATTEMPTS -->
                <div id="section-store-attempts" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-6 flex items-center gap-2"><i class="fas fa-eye text-cyan-500"></i> Store Attempts (Recent)</h4>
                        <p class="text-sm text-slate-500 mb-6">These are attempts from your storefront where customers initiated payment but may not have completed the order. You cannot force delivery from here.</p>
                        <div id="store-attempts-container" class="space-y-3"></div>
                    </div>
                </div>

                <!-- PAYMENT HISTORY (v4.2.0) -->
                <!-- PAYMENT HISTORY & WITHDRAWALS -->
                <div id="section-payment-history" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200 mb-8">
                        <h4 class="text-xl font-semibold mb-6 flex items-center gap-2"><i class="fas fa-clock-rotate-left text-indigo-500"></i> Payment Sessions</h4>
                        <p class="text-sm text-slate-500 mb-6">Your recent Paystack payment sessions.</p>
                        <div id="payment-sessions-container" class="space-y-3"></div>
                    </div>
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-6 flex items-center gap-2"><i class="fas fa-hand-holding-dollar text-amber-500"></i> Withdrawal History</h4>
                        <div id="withdrawal-history-container"></div>
                    </div>
                </div>

                <!-- TIER UPGRADE -->
                <?php if ( get_option( 'highest_enable_tier_upgrade', 'no' ) === 'yes' && $this->get_parent_permission( $user_id, 'allow_upgrade' ) && ! $this->subagent_has_wholesale_enabled( $user_id ) ) : ?>
                <div id="section-tier-upgrade" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2">
                            <i class="fas fa-arrow-up text-purple-500"></i> Upgrade Your Tier
                        </h4>
                        <p class="text-sm text-slate-500 mb-6">Pay a one-time fee to move to a higher tier and enjoy lower cost prices.</p>
                        <div id="tier-upgrade-container"><p class="text-slate-400 italic">Loading available upgrades…</p></div>
                    </div>
                </div>
                <?php endif; ?>

                <div id="section-profile" class="section hidden"><div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200"><h4 class="text-xl font-semibold mb-6 flex items-center gap-2"><i class="fas fa-user-circle text-indigo-500"></i> My Profile</h4><form id="profile-form" class="space-y-5"><div><label class="block text-sm font-medium text-slate-600 mb-2">Full Name</label><input type="text" id="profile-name" value="<?php echo esc_attr( wp_get_current_user()->display_name ); ?>" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none" required></div><div><label class="block text-sm font-medium text-slate-600 mb-2">Phone Number</label><input type="tel" id="profile-phone" value="<?php echo esc_attr( get_user_meta( $user_id, 'phone', true ) ); ?>" maxlength="10" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none" required></div><div class="border-t border-slate-100 pt-4 mt-2"><p class="text-sm font-bold text-slate-700 mb-4">📱 Mobile Money Details (for automatic withdrawals)</p><div><label class="block text-sm font-medium text-slate-600 mb-2">MoMo Provider</label><select id="profile-momo-provider" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none"><option value="">-- Select Provider --</option><option value="mtn" <?php selected( get_user_meta( $user_id, 'highest_momo_provider', true ), 'mtn' ); ?>>MTN Mobile Money</option><option value="telecel" <?php selected( get_user_meta( $user_id, 'highest_momo_provider', true ), 'telecel' ); ?>>Telecel Cash (Vodafone Cash)</option><option value="airteltigo" <?php selected( get_user_meta( $user_id, 'highest_momo_provider', true ), 'airteltigo' ); ?>>AirtelTigo Money</option></select></div><div><label class="block text-sm font-medium text-slate-600 mb-2">MoMo Phone Number</label><input type="tel" id="profile-momo-phone" value="<?php echo esc_attr( get_user_meta( $user_id, 'highest_momo_phone', true ) ); ?>" maxlength="10" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none" placeholder="e.g. 0551234567"></div></div><div><label class="block text-sm font-medium text-slate-600 mb-2">Email Address</label><input type="email" id="profile-email" value="<?php echo esc_attr( wp_get_current_user()->user_email ); ?>" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none" required></div><div><label class="block text-sm font-medium text-slate-600 mb-2">New Password</label><input type="password" id="profile-password" placeholder="Min 8 characters" class="w-full p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-[#FFD700]/20 outline-none"></div><div><label class="flex items-center gap-2"><input type="checkbox" id="profile-sms-disabled" <?php checked( $this->is_agent_sms_disabled( $user_id ) ); ?>> Disable SMS notifications</label></div><button type="submit" class="w-full py-5 bg-indigo-600 hover:bg-indigo-700 text-white font-bold text-base rounded-3xl transition-all flex items-center justify-center gap-2">Update Profile</button></form></div></div>

                <div id="section-waec-tracker" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2"><i class="fas fa-search text-purple-500"></i> Retrieve Customer Checker</h4>
                        <p class="text-sm text-slate-500 mb-6">Look up a customer's WAEC checker by their phone number or email.</p>
                        <div class="flex flex-col sm:flex-row gap-4">
                            <input type="text" id="waec-search-agent" placeholder="Phone or Email" class="flex-1 p-5 bg-slate-50 border border-slate-200 rounded-3xl text-xl focus:ring-4 focus:ring-purple-500/20 outline-none">
                            <button onclick="searchWaecAgent()" class="bg-purple-600 text-white px-8 py-5 rounded-3xl font-bold">Search</button>
                        </div>
                        <div id="waec-result-agent" class="mt-6"></div>
                    </div>
                </div>

                <!-- REFERRALS SECTION -->
                <!-- REFERRALS (ORIGINAL) -->
                <div id="section-referrals" class="section hidden">
                    <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                        <h4 class="text-xl font-semibold mb-4 flex items-center gap-2"><i class="fas fa-users text-indigo-500"></i> My Sub-agents &amp; Referral Bonus</h4>
                        <div id="referral-link-box" class="mb-6 p-4 bg-indigo-50 rounded-2xl">
                            <p class="font-medium text-indigo-800">Your Referral Link:</p>
                            <input type="text" readonly id="referral-link-input" value="<?php echo esc_url( home_url( '/?ref=' . $user_id ) ); ?>" class="w-full p-3 bg-white border rounded-xl text-sm" onclick="this.select();document.execCommand('copy');alert('✅ Link copied!')">
                            <p class="text-xs text-slate-500 mt-2">Share this link – when someone registers with it, they become your sub-agent forever.</p>
                        </div>
                        <div class="mb-6">
                            <label class="block text-sm font-medium mb-2">Referral Bonus (% of sub-agent's profit)</label>
                            <input type="number" id="referral-bonus-input" step="1" min="0" max="<?php echo intval( get_option( 'highest_referral_bonus_limit_pct', 20 ) ); ?>" class="w-full p-5 bg-slate-50 border rounded-3xl" placeholder="e.g. 5">
                            <p class="text-xs text-slate-500 mt-2">You earn this percentage of each sale's profit made by your sub-agents. (Max allowed: <?php echo intval( get_option( 'highest_referral_bonus_limit_pct', 20 ) ); ?>%)</p>
                            <button onclick="saveReferralBonus()" class="mt-4 bg-indigo-600 text-white px-6 py-3 rounded-2xl font-bold">SAVE BONUS</button>
                        </div>
                        <button onclick="document.getElementById('add-subagent-modal').classList.remove('hidden')" class="mb-6 bg-indigo-600 text-white px-6 py-3 rounded-2xl font-bold">
                            <i class="fas fa-user-plus mr-2"></i> Add Sub-Agent
                        </button>

                        <!-- Add Sub-Agent Modal -->
                        <div id="add-subagent-modal" class="hidden fixed inset-0 bg-black/70 flex items-center justify-center z-[9999] p-4">
                            <div class="bg-white rounded-3xl p-8 w-full max-w-md shadow-2xl">
                                <div class="flex justify-between items-center mb-6">
                                    <h3 class="text-2xl font-semibold text-gray-800">Register Sub-Agent</h3>
                                    <button onclick="document.getElementById('add-subagent-modal').classList.add('hidden')" class="text-gray-400 hover:text-gray-600 text-2xl">&times;</button>
                                </div>
                                <form id="subagent-form" class="space-y-5">
                                    <input type="text" id="sub-name" placeholder="Full Name" required class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]">
                                    <input type="tel" id="sub-phone" placeholder="Phone (0551234567)" maxlength="10" required class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]">
                                    <input type="email" id="sub-email" placeholder="Email Address" required class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]">
                                    <input type="password" id="sub-password" placeholder="Password (min 8 chars)" required class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F]">
                                    <button type="submit" id="subagent-submit-btn" class="w-full py-4 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-semibold rounded-3xl transition flex items-center justify-center">
                                        Create Sub-Agent Account
                                    </button>
                                </form>
                            </div>
                        </div>

                        <div id="subagents-container"></div>

                        <!-- MATURATION PANEL -->
                        <div class="mt-8 bg-amber-50 border border-amber-200 p-6 rounded-2xl">
                            <h4 class="text-lg font-semibold mb-4 flex items-center gap-2"><i class="fas fa-lock text-amber-600"></i> Locked Referral Bonuses</h4>
                            <div class="mb-4">
                                <p class="text-sm">Total locked: <strong id="locked-total-display">GH₵0.00</strong></p>
                                <p class="text-sm">Claimable when total reaches: <strong id="threshold-display">GH₵<?php echo number_format( (float) get_option( 'highest_referral_maturation_threshold', 50 ), 2 ); ?></strong></p>
                            </div>
                            <button onclick="claimLockedBonuses()" id="claim-locked-btn" class="bg-emerald-600 text-white px-6 py-3 rounded-2xl font-bold disabled:opacity-50 opacity-50" disabled>
                                <i class="fas fa-hand-holding-usd mr-2"></i> Claim Now
                            </button>
                            <p class="text-xs text-slate-500 mt-2">Click to move all locked bonuses to your withdrawable profit. You can only claim once the threshold is reached.</p>
                        </div>

                        <!-- WHOLESALE PRICES -->
                        <?php if ( get_option( 'highest_enable_wholesale_pricing', 'no' ) === 'yes' && $this->get_parent_permission( $user_id, 'allow_wholesale' ) ) : ?>
                        <div class="mt-8 bg-blue-50 border border-blue-200 p-6 rounded-2xl">
                            <h4 class="text-lg font-semibold mb-4 flex items-center gap-2">
                                <i class="fas fa-tags text-blue-600"></i> Wholesale Prices for Sub-Agents
                            </h4>
                            <p class="text-sm text-slate-600 mb-4">Set the price your sub-agents will pay for each package. They keep the profit above this price.</p>
                            <div id="wholesale-editor" class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 text-sm"></div>
                            <button onclick="saveWholesalePrices()" class="mt-4 bg-blue-600 text-white px-6 py-3 rounded-2xl font-bold">
                                <i class="fas fa-save mr-2"></i> Save Wholesale Prices
                            </button>
                        </div>
                        <?php endif; ?>
                    </div>
                </div>
            </main>
            <nav class="lg:hidden fixed bottom-0 left-0 right-0 bg-white border-t shadow-2xl z-50">
    <div class="flex justify-around items-center py-1.5 px-1 text-[#001F3F] overflow-x-auto">
        <a onclick="switchSection('dashboard')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-house text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Home</span>
        </a>
        <a onclick="switchSection('buy')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-bolt text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Buy</span>
        </a>
        <a onclick="switchSection('wallet')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-wallet text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Wallet</span>
        </a>
        <a onclick="switchSection('prices')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-tags text-sm text-purple-500"></i><span class="text-[9px] mt-0.5 leading-tight text-purple-500">Prices</span>
        </a>
        <a onclick="switchSection('storefront')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-store text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Store</span>
        </a>
        <a onclick="switchSection('afa')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-id-card text-sm text-purple-500"></i><span class="text-[9px] mt-0.5 leading-tight text-purple-500">AFA</span>
        </a>
        <a onclick="switchSection('withdraw')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-hand-holding-dollar text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Withdraw</span>
        </a>
        <a onclick="switchSection('transactions')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-list-ul text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Orders</span>
        </a>
        <a onclick="switchSection('payment-history')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-clock-rotate-left text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Payments</span>
        </a>
        <a onclick="switchSection('store-attempts')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-eye text-sm text-cyan-500"></i><span class="text-[9px] mt-0.5 leading-tight text-cyan-500">Attempts</span>
        </a>
        <a onclick="switchSection('topup-history')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-history text-sm text-blue-500"></i><span class="text-[9px] mt-0.5 leading-tight text-blue-500">History</span>
        </a>
        <a onclick="switchSection('profile')" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-user-circle text-sm"></i><span class="text-[9px] mt-0.5 leading-tight">Profile</span>
        </a>
        <a href="https://test.highestdataplug.com" target="_blank" class="flex flex-col items-center py-1 px-1 active:scale-95 transition min-w-[45px]">
            <i class="fas fa-store text-sm text-yellow-500"></i><span class="text-[9px] mt-0.5 leading-tight text-yellow-500">Market</span>
        </a>
    </div>
            </nav>
            <a href="https://wa.me/<?php echo esc_attr( str_replace( '+', '', $whatsapp ) ); ?>?text=Hi%20Highest%20Data%20Plug%20support%2C%20I%20need%20help" target="_blank" class="fixed bottom-6 right-6 bg-[#25D366] text-white w-14 h-14 rounded-3xl flex items-center justify-center shadow-2xl hover:scale-110 transition text-3xl z-[99999]"><i class="fab fa-whatsapp"></i></a>
            <div id="agent-buy-modal" class="hidden fixed inset-0 bg-black/70 flex items-center justify-center z-[9999] p-4">
                <div class="bg-white rounded-3xl p-8 w-full max-w-md shadow-2xl">
                    <div class="flex justify-between items-center mb-6"><h3 class="text-2xl font-semibold text-gray-800">Customer Telephone Number</h3><button onclick="closeAgentBuyModal()" class="text-gray-400 hover:text-gray-600 text-2xl">×</button></div>
                    <div class="space-y-5">
                        <input type="tel" id="agent-modal-phone" placeholder="0551234567" maxlength="10" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F] text-lg" required>
                        <div class="text-xs text-amber-600 flex items-center gap-1"><i class="fas fa-info-circle"></i> Verify your Ghana number (10 digits only)</div>
                        <input type="email" id="agent-modal-email" placeholder="Email (optional - for Paystack receipt)" class="w-full p-4 border border-gray-300 rounded-3xl focus:outline-none focus:border-[#001F3F] text-lg">
                        <div class="text-xs text-amber-600 flex items-center gap-1"><i class="fas fa-exclamation-triangle"></i> ⚠️ Please enter your email to avoid payment failure.</div>
                        <button onclick="submitAgentBuyPhone()" class="w-full py-4 bg-[#FFD700] hover:bg-yellow-400 text-[#001F3F] font-semibold rounded-3xl transition flex items-center justify-center"><i class="fas fa-credit-card mr-2"></i> Continue to Secure Payment</button>
                        <button onclick="submitAgentBuyWithWallet()" class="w-full py-4 mt-3 bg-emerald-600 hover:bg-emerald-700 text-white font-semibold rounded-3xl transition flex items-center justify-center"><i class="fas fa-wallet mr-2"></i> Use Wallet &amp; Deduct Immediately</button>
                    </div>
                </div>
            </div>
        </div>
        <script>
        const EMERGENCY_PROMPT_ENABLED = <?php echo $this->emergency_prompt_enabled ? 'true' : 'false'; ?>;
    
    // ─── PWA Install (reliable) ──────────────────────────────
let deferredPrompt;

window.addEventListener('beforeinstallprompt', (e) => {
    // Prevent the mini-infobar from appearing on mobile
    e.preventDefault();
    // Save the event for later use
    deferredPrompt = e;
    // Show the install button
    const btn = document.getElementById('pwa-install-btn');
    if (btn) btn.classList.remove('hidden');
});

function installApp() {
    if (deferredPrompt) {
        // Native install prompt is available – trigger it
        deferredPrompt.prompt();
        deferredPrompt.userChoice.then((choiceResult) => {
            if (choiceResult.outcome === 'accepted') {
                alert('✅ App installed! You can now open it from your home screen.');
            }
            deferredPrompt = null;
            // Hide the button after the choice
            const btn = document.getElementById('pwa-install-btn');
            if (btn) btn.classList.add('hidden');
        });
    } else {
        // No native prompt available – give manual instructions
        const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream;
        if (isIOS) {
            alert('📱 To install this app on your iPhone:\n\n1. Tap the Share button (square with arrow)\n2. Scroll down and tap "Add to Home Screen"\n3. Tap "Add"');
        } else {
            alert('📲 To install this app:\n\nOpen your browser menu (⋮) and select "Add to Home Screen" or "Install app".');
        }
    }
}

// Optional: hide the button if the app is already running in standalone mode
if (window.matchMedia('(display-mode: standalone)').matches) {
    const btn = document.getElementById('pwa-install-btn');
    if (btn) btn.classList.add('hidden');
}
const agentBundles = <?php echo json_encode( $agent_bundles ); ?>;
        const disabledPackages = <?php echo json_encode( get_option( 'highest_disabled_packages', array() ) ); ?>;
        let currentAgentPackage = null;
        function renderAgentBuyGrid() {
            let html = '';
            agentBundles.forEach(pkg => {
                if (disabledPackages.includes(pkg.key)) return;
                const colorClass = pkg.network === 'MTN' ? 'text-[#FFD700]' : (pkg.network === 'TELECEL' ? 'text-red-500' : 'text-blue-500');
                const btnClass = pkg.network === 'MTN' ? 'bg-[#FFD700] text-[#001F3F] hover:bg-yellow-400' : (pkg.network === 'TELECEL' ? 'bg-red-600 text-white hover:bg-red-700' : 'bg-blue-600 text-white hover:bg-blue-700');
                html += `<div onclick="startAgentBuy('${pkg.key}',${pkg.price},${pkg.network_id},${pkg.shared_bundle})" class="bg-white border-2 border-slate-200 hover:border-[#001F3F] p-4 md:p-5 rounded-3xl flex flex-col min-h-[170px] transition-all hover:shadow-2xl cursor-pointer"><div class="flex flex-col"><p class="text-xs uppercase font-bold ${colorClass}">${pkg.network}</p><p class="text-2xl leading-none font-bold text-slate-800 mt-1">${pkg.size}</p><div class="mt-4 text-lg font-bold text-emerald-600">GH₵${pkg.price.toFixed(2)}</div></div><button class="mt-auto w-full py-3 ${btnClass} font-bold rounded-3xl transition-all text-sm">BUY NOW</button></div>`;
            });
            document.getElementById('agent-buy-grid').innerHTML = html;
        }
        function startAgentBuy(key, price, network_id, shared_bundle) {
            <?php if ( $disable_agent_buy || $disable_all_buy ) : ?>alert("Buying is currently disabled by admin."); return;<?php endif; ?>
            if (!PAYSTACK_PUBLIC_KEY) { alert("Paystack not configured."); return; }
            currentAgentPackage = { key, price, network_id, shared_bundle };
            showAgentBuyModal();
        }
        function showAgentBuyModal() { const m = document.getElementById('agent-buy-modal'); m.classList.remove('hidden'); m.classList.add('flex'); const pi = document.getElementById('agent-modal-phone'); pi.value = document.getElementById('agent-buy-phone').value.trim() || ''; pi.focus(); }
        function closeAgentBuyModal() { const m = document.getElementById('agent-buy-modal'); m.classList.add('hidden'); m.classList.remove('flex'); }
        function submitAgentBuyPhone() {
            const phone = document.getElementById('agent-modal-phone').value.trim();
            if (!phone) { alert("⚠️ Please enter the customer's telephone number"); return; }
            const emailInput = document.getElementById('agent-modal-email').value.trim();
            const email = emailInput ? emailInput : phone + '@highestdataplug.com';
            closeAgentBuyModal();
            const pkg = currentAgentPackage;
            if (!pkg) return;
            const paystackRef = 'AGENT-' + Date.now() + '-' + Math.floor(Math.random() * 1000000);
            const logData = new FormData();
            logData.append('action', 'highest_log_payment_initiation');
            logData.append('nonce', HIGHEST_NONCE);
            logData.append('ref', paystackRef);
            logData.append('amount', pkg.price);
            logData.append('phone', phone);
            logData.append('email', email);
            logData.append('package_name', pkg.key);
            logData.append('network_id', pkg.network_id);
            logData.append('shared_bundle', pkg.shared_bundle);
            logData.append('selling_price', pkg.price);
            logData.append('agent_id', '<?php echo $user_id; ?>');
            logData.append('order_type', 'agent');
            fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: logData }).then(r => r.json()).then(res => console.log('Payment session logged:', res)).catch(e => console.error('Payment session log error:', e));
            var gateway = "<?php echo esc_js( get_option('highest_payment_gateway','paystack') ); ?>";
            if (gateway === 'moolre') {
                var mfd = new FormData();
                mfd.append('action', 'highest_get_moolre_checkout_url');
                mfd.append('nonce', HIGHEST_NONCE);
                mfd.append('ref', paystackRef);
                mfd.append('amount', pkg.price);
                mfd.append('phone', phone);
                mfd.append('email', email);
                mfd.append('description', pkg.key + ' to ' + phone);
                fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: mfd })
                .then(r => r.json())
                .then(res => {
                    if (res.success) { window.location.href = res.data.url; }
                    else { alert("❌ Payment error: " + res.data); }
                });
                return;
            }
            const handler = PaystackPop.setup({
                key: PAYSTACK_PUBLIC_KEY,
                email: email,
                amount: Math.round(pkg.price * 100),
                currency: "GHS",
                ref: paystackRef,
                callback: function(response) {
                    const fd = new FormData();
                    fd.append('action', 'highest_buy_data_verified');
                    fd.append('reference', response.reference);
                    fd.append('trans', response.trans);
                    fd.append('nonce', HIGHEST_NONCE);
                    fd.append('phone', phone);
                    fd.append('email', email);
                    fd.append('package_name', pkg.key);
                    fd.append('network_id', pkg.network_id);
                    fd.append('shared_bundle', pkg.shared_bundle);
                    fd.append('selling_price', pkg.price);
                    fd.append('agent_id', '<?php echo $user_id; ?>');
                    fd.append('order_type', 'agent');
                    fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: fd }).then(r => r.json()).then(res => {
                        var agentMsg = <?php echo json_encode( $this->get_delivery_message( 'highest_agent_success_msg', '✅ Data purchased successfully! Delivery usually takes about {minutes} minute(s).' ) ); ?>; if (res.success) { alert(agentMsg); loadOrders(); location.reload(); }
                        else { handleAgentOrderFailure(res.data || "Request failed", pkg); }
                    }).catch(() => {
                        fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: fd }).then(r => r.json()).then(res => {
                            var agentMsg = <?php echo json_encode( $this->get_delivery_message( 'highest_agent_success_msg', '✅ Data purchased successfully! Delivery usually takes about {minutes} minute(s).' ) ); ?>; if (res.success) { alert(agentMsg); loadOrders(); location.reload(); }
                            else { handleAgentOrderFailure(res.data || "Request failed", pkg); }
                        }).catch(() => handleAgentOrderFailure("Network error", pkg));
                    });
                },
                onClose: function() {}
            });
            handler.openIframe();
        }
        function submitAgentBuyWithWallet() {
            const phone = document.getElementById('agent-modal-phone').value.trim();
            if (!phone) { alert("⚠️ Please enter the customer's telephone number"); return; }
            closeAgentBuyModal();
            const pkg = currentAgentPackage;
            if (!pkg) return;
            const fd = new FormData();
            fd.append('action', 'highest_agent_buy_data');
            fd.append('nonce', HIGHEST_NONCE);
            fd.append('package_key', pkg.key);
            fd.append('phone', phone);
            fd.append('selling_price', pkg.price);
            fd.append('network_id', pkg.network_id);
            fd.append('shared_bundle', pkg.shared_bundle);
            fd.append('payment_method', 'wallet');
            fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: fd })
            .then(r => r.json())
            .then(res => {
                var agentWMsg = <?php echo json_encode( $this->get_delivery_message( 'highest_agent_success_msg', '✅ Data purchased successfully! Delivery usually takes about {minutes} minute(s).' ) ); ?>; if (res.success) { alert(agentWMsg); loadOrders(); location.reload(); }
                else { handleAgentOrderFailure(res.data || "Request failed", pkg); }
            })
            .catch(() => handleAgentOrderFailure("Network error", pkg));
        }
        function handleAgentOrderFailure(errorMsg, pkg) {
            if (EMERGENCY_PROMPT_ENABLED && confirm("❌ " + errorMsg + "\n\nDo you want to send an emergency SMS to admin to investigate?")) {
                var emsg = prompt("Add details (optional):", pkg ? "Package: " + pkg.key + ", amount: GH₵" + pkg.price : "");
                if (emsg !== null) {
                    var fdSOS = new FormData();
                    fdSOS.append('action', 'highest_send_emergency_sms');
                    fdSOS.append('nonce', HIGHEST_NONCE);
                    fdSOS.append('message', emsg || errorMsg);
                    fetch(HIGHEST_AJAX, { method:'POST', credentials:'same-origin', body:fdSOS })
                    .then(r => r.json())
                    .then(sosRes => {
                        if (sosRes.success) alert("✅ Emergency SMS sent!");
                        else alert("❌ Failed to send SMS: " + (sosRes.data || "Try again."));
                    })
                    .catch(() => alert("Network error."));
                }
            } else {
                alert("❌ " + errorMsg + " Please try again or contact admin.");
            }
        }
        function renderPriceEditor() {
            let html = '';
            agentBundles.forEach(pkg => {
                html += `<div class="bg-slate-50 border border-slate-200 p-5 rounded-3xl flex flex-col"><div class="flex justify-between items-start"><div><p class="text-xs uppercase font-bold text-slate-400">${pkg.network}</p><p class="text-2xl font-bold text-slate-800">${pkg.size}</p></div><div class="text-right"><input type="number" step="0.01" min="${pkg.min_price.toFixed(2)}" value="${pkg.price.toFixed(2)}" id="price-input-${pkg.key}" data-key="${pkg.key}" class="w-28 text-right font-bold text-3xl bg-white border border-slate-300 rounded-2xl px-4 py-3 focus:outline-none focus:border-purple-500 text-emerald-700"><button onclick="saveSinglePrice('${pkg.key}')" class="mt-3 text-xs bg-purple-600 hover:bg-purple-700 text-white px-5 py-2 rounded-2xl font-medium">SAVE INSTANTLY</button></div></div><small class="text-xs text-slate-400 mt-3">Min GH₵${(pkg.min_price + 0.1).toFixed(2)} • Profit = selling − ${pkg.min_price.toFixed(2)}</small></div>`;
            });
            document.getElementById('price-editor-mobile').innerHTML = html;
            document.getElementById('price-editor').innerHTML = html;
        }
        function saveSinglePrice(key) {
            const input = document.getElementById('price-input-' + key);
            if (!input) return;
            const value = parseFloat(input.value);
            if (isNaN(value)) { alert("Invalid price"); return; }
            const fd = new FormData();
            fd.append('action', 'highest_save_agent_prices');
            fd.append('nonce', HIGHEST_NONCE);
            fd.append(key, value);
            fetch(HIGHEST_AJAX, { method: 'POST', credentials: 'same-origin', body: fd }).then(r => r.json()).then(res => {
                if (res.success) { alert("✅ " + key + " price saved!"); location.reload(); }
                else { alert("❌ " + (res.data || "Failed to save price")); }
            });
        }
        function toggleMobileMenu() { document.getElementById('agent-sidebar').classList.toggle('-translate-x-full'); }
        function switchSection(section) {
            document.querySelectorAll('.section').forEach(s => s.classList.add('hidden'));
            document.getElementById('section-' + section).classList.remove('hidden');
            document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active'));
            if (section === 'store-attempts') loadStoreAttempts();
            if (section === 'payment-history') { loadPaymentSessions(); loadWithdrawalHistory(); }
            if (section === 'referrals') { loadReferrals(); loadLockedBonusStatus(); loadWholesalePrices(); }
            if (section === 'tier-upgrade') loadTierUpgrades();
            if (section === 'waec-tracker') { document.getElementById('waec-result-agent').innerHTML = ''; }
        }

        function searchWaecAgent() {
            var container = document.getElementById('waec-result-agent');
            container.innerHTML = '<p class="text-slate-500">Searching…</p>';
            var fd = new FormData();
            fd.append('action', 'highest_waec_retrieve');
            fd.append('nonce', HIGHEST_NONCE);
            fd.append('search', document.getElementById('waec-search-agent').value.trim());
            fetch(HIGHEST_AJAX, { method:'POST', credentials:'same-origin', body:fd })
            .then(r => r.json())
            .then(res => {
                if (res.success && res.data) {
                    var o = res.data;
                    container.innerHTML = `<div class="bg-green-50 border border-green-200 p-4 rounded-2xl">
                        <p class="font-bold text-green-800">${o.exam_type} Result Checker</p>
                        <p><strong>Serial:</strong> ${o.serial_number}</p>
                        <p><strong>PIN:</strong> ${o.pin}</p>
                        <p class="text-xs text-slate-500 mt-2">Sent: ${o.pin_sent_at}</p>
                    </div>`;
                } el
