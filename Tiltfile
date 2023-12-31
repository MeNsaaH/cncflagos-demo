load('ext://dotenv', 'dotenv')
load('ext://helm_remote', 'helm_remote')
load('ext://namespace', 'namespace_yaml')
load('ext://uibutton', 'cmd_button')
load('./.tilt/terraform/Tiltfile', 'local_terraform_resource')
load('./.tilt/utils/Tiltfile', 'check_env_set')
dotenv()

config.define_bool("enable_repo", True, 'create a new project for testing this app')
config.define_string("vcs-type")
config.define_bool("live_debug") # not used, but kept for backwards compat
config.define_string("ngrok_fqdn")
cfg = config.parse()

allow_k8s_contexts([
  'kind-kind',
  'docker-desktop',
  'minikube',
])

k8s_namespace='kubechecks'
k8s_yaml(namespace_yaml(k8s_namespace), allow_duplicates=False)
k8s_resource(
  objects=['kubechecks:namespace'],
  labels=["localdev"],
  new_name='k8s:namespace'
)
k8s_context=k8s_context()

# start Tilt with no enabled resources
# config.clear_enabled_resources()

# /////////////////////////////////////////////////////////////////////////////
# N G R O K
# /////////////////////////////////////////////////////////////////////////////

# Load NGROK Tiltfile
load('./localdev/ngrok/Tiltfile', 'deploy_ngrok', 'get_ngrok_url')
deploy_ngrok(cfg)

# /////////////////////////////////////////////////////////////////////////////
# A R G O  C D
# /////////////////////////////////////////////////////////////////////////////

# Load ArgoCD Tiltfile
load('./localdev/argocd/Tiltfile', 'deploy_argo')
deploy_argo()

#load('./localdev/reloader/Tiltfile', 'deploy_reloader')
#deploy_reloader()

# /////////////////////////////////////////////////////////////////////////////
# T E R R A F O R M
# /////////////////////////////////////////////////////////////////////////////

tfcOutputs=local_terraform_resource(
  'tf-random-pet',
  dir='./localdev/terraform',
  deps=[
    'localdev/terraform/*.tf',
  ],
  labels=["terraform"],
)

kubeProject=""
if cfg.get('enable_repo', True):
  if cfg.get('vcs-type') == 'gitlab':
    check_env_set("GITLAB_TOKEN")

    gitlabOutputs=local_terraform_resource(
      'tf-gitlab',
      dir='./localdev/terraform/gitlab',
      env={
        'GITLAB_TOKEN': os.getenv('GITLAB_TOKEN'),
        'TF_VAR_ngrok_url': get_ngrok_url(cfg),
        'TF_VAR_kubechecks_gitlab_hook_secret_key': os.getenv('KUBECHECKS_WEBHOOK_SECRET') if os.getenv('KUBECHECKS_WEBHOOK_SECRET') != None else "",
      },
      deps=[
        './localdev/terraform/*.tf',
        './localdev/terraform/terraform.tfstate',
        './localdev/terraform/gitlab/*.tf',
      ],
      resource_deps=[
        'tf-random-pet',
        'ngrok'
      ],
      labels=['terraform']
    )
    kubeProject=gitlabOutputs.setdefault('gitlab_project_name', '') if gitlabOutputs else 'foo'
    watch_file('./localdev/terraform/gitlab/project.url')
  else:
    check_env_set("GITHUB_TOKEN")

    githubOutputs=local_terraform_resource(
      'tf-github',
      dir='./localdev/terraform/github',
      env={
        'GITHUB_TOKEN': os.getenv('GITHUB_TOKEN'),
        'TF_VAR_ngrok_url': get_ngrok_url(cfg),
        'TF_VAR_kubechecks_github_hook_secret_key': os.getenv('KUBECHECKS_WEBHOOK_SECRET') if os.getenv('KUBECHECKS_WEBHOOK_SECRET') != None else "",
      },
      deps=[
        './localdev/terraform/*.tf',
        './localdev/terraform/terraform.tfstate',
        './localdev/terraform/github/*.tf',
      ],
      resource_deps=[
        'tf-random-pet',
        'ngrok'
      ],
      labels=['terraform']
    )
    kubeProject=githubOutputs.setdefault('github_repo_name', '') if githubOutputs else 'foo'
    watch_file('./localdev/terraform/github/project.url')

# /////////////////////////////////////////////////////////////////////////////
# K U B E C H E C K S
# /////////////////////////////////////////////////////////////////////////////

k8s_yaml(helm(
  './charts/kubechecks/',
  namespace='kubechecks',
  name='kubechecks',
  values='./localdev/kubechecks/values.yaml',
  set=['deployment.env.KUBECHECKS_WEBHOOK_URL_BASE=' + get_ngrok_url(cfg), 'deployment.env.NGROK_URL=' + get_ngrok_url(cfg),
        'deployment.env.KUBECHECKS_ARGOCD_WEBHOOK_URL='+ get_ngrok_url(cfg) +'/argocd/api/webhook',
        'deployment.env.KUBECHECKS_ENABLE_CONFTEST=true',
        'deployment.env.KUBECHECKS_VCS_TYPE=' + cfg.get('vcs-type', 'gitlab'),
        'secrets.env.KUBECHECKS_VCS_TOKEN=' + (os.getenv('GITLAB_TOKEN') if 'gitlab' in cfg.get('vcs-type', 'gitlab') else os.getenv('GITHUB_TOKEN')),
        'secrets.env.KUBECHECKS_WEBHOOK_SECRET=' + (os.getenv('KUBECHECKS_WEBHOOK_SECRET') if os.getenv('KUBECHECKS_WEBHOOK_SECRET') != None else ""),
        'secrets.env.KUBECHECKS_OPENAI_API_TOKEN=' + (os.getenv('OPENAI_API_TOKEN') if os.getenv('OPENAI_API_TOKEN') != None else ""),],
))

k8s_resource(
  'kubechecks',
  port_forwards=['2345:2345', '8080:8080'],
  resource_deps=[
    'k8s:namespace'
  ],
  labels=["kubechecks"]
)

k8s_resource(
    objects=[
      'kubechecks-argocd-application-controller:clusterrole',
      'kubechecks-argocd-server:clusterrole',
      'kubechecks-argocd-application-controller:clusterrolebinding',
      'kubechecks-argocd-server:clusterrolebinding',
    ],
    new_name='kubechecks-rbac',
    labels=["kubechecks"],
    resource_deps=['k8s:namespace']
)

helm_remote(
    'reloader',
    repo_url='https://stakater.github.io/stakater-charts',
    release_name='reloader',
    namespace='kubechecks',
    version='1.0.26'
)

load("localdev/test_apps/Tiltfile", "install_test_apps")
install_test_apps(cfg)

load("localdev/test_appsets/Tiltfile", "install_test_appsets")
install_test_appsets(cfg)
