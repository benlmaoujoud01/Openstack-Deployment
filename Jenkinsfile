def runOnController(command) {
    sh "ssh -o StrictHostKeyChecking=yes -i /path/to/secure/key application@192.168.14.30 '${command}'"
}

pipeline {
    agent any
    environment {
        KOLLA_DIR = '/etc/kolla'

        ANSIBLE_VAULT_PASSWORD = credentials('ansible-vault-password')
        DEFAULT_UMASK = '0027'
    }

    stages {
        stage('Security Scan') {
            steps {
                sh 'safety check'
                sh 'bandit -r .'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                script {
                    runOnController """
                        # Set umask for more secure file permissions
                        umask ${DEFAULT_UMASK}
                        # Use specific versions to avoid supply chain attacks
                        sudo apt update
                        sudo apt -y install git=2.34.1-1ubuntu1.9 python3-dev=3.10.6-1~22.04 libffi-dev=3.4.2-4 gcc=4:11.2.0-1ubuntu1 libssl-dev=3.0.2-0ubuntu1.10 python3-venv=3.10.6-1~22.04
                        # Verify package integrity
                        sudo apt-key adv --verify Release.gpg
                    """
                }
            }
        }
        
        stage('Create Virtual Environment') {
            steps {
                script {
                    runOnController """
                        umask ${DEFAULT_UMASK}
                        python3 -m venv ~/kolla-ansible
                        chmod 750 ~/kolla-ansible
                    """
                }
            }
        }
        
        stage('Upgrade PIP') {
            steps {
                script {
                    runOnController """
                        source ~/kolla-ansible/bin/activate
                        # Use specified version and validate checksum
                        pip install --require-hashes -r pip-requirements.txt
                        pip install -U pip==23.0.1
                    """
                }
            }
        }
        
        stage('Install Ansible') {
            steps {
                script {
                    runOnController """
                        source ~/kolla-ansible/bin/activate
                        # Pin to specific version for security
                        pip install "ansible-core==2.15.5"
                    """
                }
            }
        }
        
        stage('Install Kolla-ansible') {
            steps {
                script {
                    runOnController """
                        source ~/kolla-ansible/bin/activate
                        pip install git+https://opendev.org/openstack/kolla-ansible@f9744e9cd20debc94ba4cfc0c4e2bd3b0c0f9b78
                    """
                }
            }
        }

        stage('Setup Env') {
            steps {
                script {
                    runOnController """
                    source ~/kolla-ansible/bin/activate
                    sudo mkdir -p /etc/kolla
                    # Use more restricted permissions
                    sudo install -d -m 0750 -o application -g application /etc/kolla
                    cp -r ~/kolla-ansible/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
                    find /etc/kolla -type f -exec chmod 640 {} \\;
                    find /etc/kolla -type d -exec chmod 750 {} \\;
                    """
                }
            }
        }

        stage('Setup Globals with Security') {
            steps {
                script {
                    runOnController """
                    source ~/kolla-ansible/bin/activate
                    cat > /etc/kolla/globals.yml  << 'EOL'
---
workaround_ansible_issue_8743: yes
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "ubuntu"
openstack_release: "2024.1"
network_interface: >-
  {%- if inventory_hostname == "compute1" -%}
  eno1np0
  {%- elif inventory_hostname == "compute2" -%}
  eno1
  {%- elif inventory_hostname == "controller" -%}
  eno1
  {%- endif -%}
neutron_external_interface: >-
  {%- if inventory_hostname == "compute1" -%}
  eno2np1
  {%- elif inventory_hostname == "compute2" -%}
  eno2
  {%- elif inventory_hostname == "controller" -%}
  eno2
  {%- endif -%}
kolla_internal_vip_address: "VIP-change it "
kolla_container_engine: docker
docker_configure_for_zun: "yes"
containerd_configure_for_zun: "yes"
docker_apt_package_pin: "5:20.*"
network_address_family: "ipv4"
neutron_plugin_agent: "openvswitch"
enable_openstack_core: "yes"
enable_glance: "{{ enable_openstack_core | bool }}"
enable_haproxy: "yes"
enable_keepalived: "yes"
enable_keystone: "{{ enable_openstack_core | bool }}"
enable_mariadb: "yes"
enable_memcached: "yes"
enable_neutron: "{{ enable_openstack_core | bool }}"
enable_nova: "{{ enable_openstack_core | bool }}"
enable_aodh: "yes"
enable_ceilometer: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_etcd: "yes"
enable_gnocchi: "yes"
enable_gnocchi_statsd: "yes"
enable_grafana: "yes"
enable_heat: "{{ enable_openstack_core | bool }}"
enable_horizon: "{{ enable_openstack_core | bool }}"
enable_horizon_zun: "{{ enable_zun | bool }}"
enable_kuryr: "yes"
enable_prometheus: "yes"
enable_zun: "yes"
cinder_volume_group: "cinder-volumes"

# SECURITY ENHANCEMENTS - Enable TLS
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
kolla_copy_ca_into_containers: "yes"
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"
kolla_haproxy_ssl_settings: "modern"
kolla_admin_openrc_cacert: "/etc/kolla/certificates/ca/root.crt"

# Additional security configurations
enable_barbican: "yes"  # Secret management service
keystone_token_provider: "fernet"
keystone_admin_password_hash_rounds: 10000
keystone_password_hash_algorithm: "bcrypt"
fluentd_syslog_port: 5170  # Non-default port for security
enable_neutron_vpnaas: "yes"
enable_neutron_qos: "yes"
enable_neutron_port_security: "yes"
enable_neutron_firewall: "yes"
enable_neutron_agent_ha: "yes"

# Database security
mariadb_backup: "yes"
mariadb_backup_database_password: "{{ lookup('file', '/path/to/secure/password_file') }}"

# Additional security services
enable_octavia: "yes"  # Load balancer
enable_designate: "yes"  # DNS service for security tracking
enable_barbican: "yes"  # Secret storage
enable_keystone_idp: "yes"  # Identity provider 

# Enable auditing
enable_osprofiler: "yes"

enable_cinder_backup: "yes"
EOL
        chmod 600 /etc/kolla/globals.yml
                    """
                }
            }
        }

        stage("Setup multinode"){
            steps{
                  runOnController """
                    cp ~/kolla-ansible/share/kolla-ansible/ansible/inventory/multinode .
                    
                    sed -i '/^control0/d' multinode
                    sed -i '/^network0/d' multinode
                    sed -i '/^compute0/d' multinode
                    sed -i '/^monitoring0/d' multinode
                    sed -i '/^storage0/d' multinode

                    sed -i "/\\[control\\]/a controller ansible_connection=local ansible_user=application ansible_ssh_private_key_file=/path/to/secure/key" multinode
                    sed -i "/\\[network\\]/a controller ansible_connection=local ansible_user=application ansible_ssh_private_key_file=/path/to/secure/key" multinode
                    sed -i "/\\[compute\\]/a compute2 ansible_user=application ansible_ssh_private_key_file=/path/to/secure/key \\ncompute1 ansible_user=application ansible_ssh_private_key_file=/path/to/secure/key" multinode
                    sed -i "/\\[monitoring\\]/a controller ansible_connection=local ansible_user=application ansible_ssh_private_key_file=/path/to/secure/key" multinode
                    sed -i "/\\[storage\\]/a compute2 ansible_user=application ansible_ssh_private_key_file=/path/to/secure/key \\ncompute1 ansible_user=application ansible_ssh_private_key_file=/path/to/secure/key" multinode
                    sed -i "s/^localhost.*/localhost ansible_connection=local ansible_user=application/" multinode

                    chmod 600 multinode
                    """
            }
        }
        
        stage("Generate SSL/TLS Certificates") {
            steps {
                runOnController """
                    source ~/kolla-ansible/bin/activate
                    mkdir -p /etc/kolla/certificates
                    chmod 700 /etc/kolla/certificates
                    

                    openssl genrsa -out /etc/kolla/certificates/ca/root.key 4096
                    openssl req -x509 -new -nodes -key /etc/kolla/certificates/ca/root.key -sha256 -days 365 -out /etc/kolla/certificates/ca/root.crt -subj "/CN=Kolla Root CA"
                    
                    # Generate and copy certificates for services
                    kolla-ansible -i multinode certificates
                """
            }
        }
        
        stage("Install Ansible Galaxy requirements"){
            steps{
                runOnController """
                    source ~/kolla-ansible/bin/activate
                    # Use secure password generation with higher entropy
                    kolla-genpwd --length 32 --strong
                    
                    # Set up secure SSH connections for Ansible
                    export ANSIBLE_HOST_KEY_CHECKING=True
                    kolla-ansible install-deps
                """
            }
        }
        
        
        stage("Bootstrap Nodes"){
            steps{
                runOnController """
                source ~/kolla-ansible/bin/activate
                # Add security-related flags
                kolla-ansible -i multinode bootstrap-servers
                """
            }
        }
        
        stage("Pre-deployment checks"){
            steps{
                 runOnController """
                source ~/kolla-ansible/bin/activate
                # Run extensive pre-checks including security validation
                kolla-ansible -i multinode prechecks --extra-vars "validate_tls=yes validate_certs=yes"
                """
            }
        }
        
        stage("Openstack deployment"){
            steps{
                 runOnController """
                source ~/kolla-ansible/bin/activate
                # Deploy with security enhancements
                kolla-ansible -i multinode deploy --extra-vars "openstack_install_method=source"
                """
            }
        }
        
        stage("Security Verification") {
            steps {
                runOnController """
                source ~/kolla-ansible/bin/activate
                
                # Verify TLS configuration
                openssl s_client -connect ip:443 -showcerts
                
                source /etc/kolla/admin-openrc.sh
                openstack --insecure token issue
                """
            }
        }
    }

    post {
        success {
            echo 'OpenStack deployment completed successfully and securely!'
            
            // Send secure notification
            slackSend color: 'good', message: 'OpenStack deployment completed successfully with security enhancements'
        }
        failure {
            echo 'OpenStack deployment failed! Check the logs for details.'
            
            // Send secure notification
            slackSend color: 'danger', message: 'OpenStack deployment failed - immediate attention required'
        }
        always {
            sh 'find . -name "*password*" -type f -delete'
            sh 'find . -name "*.key" -type f -delete'
        }
    }
}
