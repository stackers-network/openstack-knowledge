# Horizon Internals

## Django Project Structure

Horizon is a Django project. The source tree has two key packages:

```
openstack_dashboard/          # The main Django project
├── settings.py               # Master Django settings (imports local_settings.py)
├── defaults.py               # All default values for Horizon settings
├── wsgi.py                   # WSGI entry point
├── urls.py                   # Root URL configuration
├── dashboards/               # All dashboard definitions
│   ├── project/              # "Project" dashboard (end-user facing)
│   │   ├── dashboard.py      # Dashboard class registration
│   │   ├── compute/          # Compute panel group
│   │   │   ├── panel.py      # Panel class registration
│   │   │   ├── urls.py
│   │   │   ├── views.py
│   │   │   ├── tables.py
│   │   │   ├── forms.py
│   │   │   └── tabs.py
│   │   ├── volumes/
│   │   ├── network/
│   │   └── ...
│   ├── admin/                # "Admin" dashboard (operator facing)
│   └── identity/             # "Identity" dashboard (Keystone management)
│
horizon/                      # Core framework library
├── base.py                   # Horizon, Dashboard, PanelGroup, Panel base classes
├── tables/                   # DataTable framework
├── forms/                    # SelfHandlingForm base class
├── tabs/                     # TabGroup, Tab base classes
├── workflows/                # Step-based wizard framework
├── middleware.py             # Request processing middleware
└── exceptions.py             # Exception handling and HTTP redirects
```

## Dashboard / PanelGroup / Panel Hierarchy

Horizon organizes the UI into a three-level hierarchy:

```
Horizon (registry)
└── Dashboard           (e.g., "Project", "Admin", "Identity")
    └── PanelGroup      (e.g., "Compute", "Network", "Storage")
        └── Panel       (e.g., "Instances", "Volumes", "Networks")
```

### Dashboard

A `Dashboard` subclass represents a top-level navigation tab (the horizontal bar at the top of the Horizon UI). Each dashboard has a slug and a list of panel groups.

```python
# openstack_dashboard/dashboards/project/dashboard.py
import horizon

class Project(horizon.Dashboard):
    name = _("Project")
    slug = "project"
    panels = (
        'compute',
        'volumes',
        'network',
        'object_store',
    )
    default_panel = 'overview'

horizon.register(Project)
```

### PanelGroup

A `PanelGroup` groups related panels under a collapsible heading in the sidebar:

```python
# openstack_dashboard/dashboards/project/compute/panel.py (actually defined in __init__.py)
import horizon
from openstack_dashboard.dashboards.project import dashboard

class ComputePanels(horizon.PanelGroup):
    slug = "compute"
    name = _("Compute")
    panels = (
        'overview',
        'instances',
        'images',
        'keypairs',
        'server_groups',
    )

dashboard.Project.register(ComputePanels)
```

### Panel

A `Panel` corresponds to a single section in the sidebar with its own URL subtree:

```python
# openstack_dashboard/dashboards/project/instances/panel.py
import horizon
from openstack_dashboard.dashboards.project import dashboard

class Instances(horizon.Panel):
    name = _("Instances")
    slug = "instances"
    # Optional: permissions required to access this panel
    permissions = ('openstack.services.compute',)
    # Policy rule to check
    policy_rules = (("compute", "os_compute_api:servers:index"),)

dashboard.Project.register(Instances)
```

## Panel Registration via the `enabled/` Directory

Third-party plugins and local customizations register themselves by placing Python files in the `enabled/` directory. Horizon reads all files matching `_*.py` in this directory at startup (sorted alphabetically) and executes them.

The naming convention uses numeric prefixes to control load order:

| Range | Purpose |
|---|---|
| `_10_*.py` | Project dashboard panels |
| `_20_*.py` | Admin dashboard panels |
| `_30_*.py` | Settings panels |
| `_40_*.py` | Identity dashboard panels |
| `_70_*.py` to `_90_*.py` | Third-party plugins |

### Example: Enabling a Third-Party Panel

```python
# /etc/openstack-dashboard/enabled/_70_heat_panel.py

# The dashboard the plugin panel should be registered with
ADD_INSTALLED_APPS = [
    'heat_dashboard',
]

# Auto-discover panel configuration from the installed app
AUTO_DISCOVER_STATIC_FILES = True
```

### Example: Disabling a Built-in Panel

```python
# /etc/openstack-dashboard/enabled/_10_remove_router.py
import sys

# Remove the router panel from the Project dashboard
REMOVE_PANEL = 'router'
REMOVE_PANEL_FROM_DASHBOARD = 'project'
```

## Views

Horizon panels use Django class-based views, extended with Horizon-specific mixins:

```python
# openstack_dashboard/dashboards/project/instances/views.py
from django.urls import reverse_lazy
from horizon import tables, tabs, exceptions
from openstack_dashboard import api
from . import tables as project_tables
from . import tabs as project_tabs

class IndexView(tables.DataTableView):
    table_class = project_tables.InstancesTable
    page_title = _("Instances")
    template_name = 'project/instances/index.html'

    def get_data(self):
        # Fetch instance list from Nova API
        try:
            instances, self._more = api.nova.server_list(self.request)
        except Exception:
            instances = []
            exceptions.handle(self.request, _('Unable to retrieve instances.'))
        return instances
```

## Tables (DataTable Framework)

The `DataTable` framework generates HTML tables with sortable columns, batch actions, and row actions declaratively:

```python
# openstack_dashboard/dashboards/project/instances/tables.py
from django.utils.translation import gettext_lazy as _
from horizon import tables

class TerminateInstance(tables.BatchAction):
    name = "terminate"
    action_type = "danger"

    @staticmethod
    def action_present(count):
        return ungettext_lazy("Terminate Instance", "Terminate Instances", count)

    @staticmethod
    def action_past(count):
        return ungettext_lazy("Scheduled termination of Instance",
                              "Scheduled termination of Instances", count)

    def action(self, request, obj_id):
        api.nova.server_delete(request, obj_id)

class InstancesTable(tables.DataTable):
    name = tables.Column("name", verbose_name=_("Name"),
                         link="horizon:project:instances:detail")
    status = tables.Column("status", verbose_name=_("Status"),
                           filters=(title, filters.replace_underscores))
    image_name = tables.Column("image_name", verbose_name=_("Image Name"))
    ip = tables.Column(get_ips, verbose_name=_("IP Address"))

    class Meta:
        name = "instances"
        verbose_name = _("Instances")
        table_actions = (LaunchLink, TerminateInstance, InstancesFilterAction)
        row_actions = (StartInstance, StopInstance, SoftRebootInstance,
                       RebootInstance, TerminateInstance)
```

## Forms (SelfHandlingForm)

Horizon forms inherit from `SelfHandlingForm`, which calls `handle()` on valid submission:

```python
from horizon import forms, exceptions
from openstack_dashboard import api

class UpdateInstanceForm(forms.SelfHandlingForm):
    instance_id = forms.CharField(widget=forms.HiddenInput())
    name = forms.CharField(label=_("Name"), max_length=255)
    description = forms.CharField(label=_("Description"), required=False)

    def handle(self, request, data):
        try:
            api.nova.server_update(request, data['instance_id'],
                                   data['name'], data['description'])
            messages.success(request, _('Instance was successfully updated.'))
            return True
        except Exception:
            exceptions.handle(request, ignore=True)
            return False
```

## Tabs

Tabs display multiple views on a detail page:

```python
from horizon import tabs

class OverviewTab(tabs.Tab):
    name = _("Overview")
    slug = "overview"
    template_name = "project/instances/_detail_overview.html"

    def get_context_data(self, request):
        return {"instance": self.tab_group.kwargs['instance']}

class LogTab(tabs.Tab):
    name = _("Log")
    slug = "log"
    template_name = "project/instances/_detail_log.html"

class InstanceDetailTabs(tabs.TabGroup):
    slug = "instance_details"
    tabs = (OverviewTab, LogTab, ConsoleTab)
    sticky = True
```

## Workflows (Multi-Step Wizards)

Workflows guide users through complex multi-step operations (e.g., Launch Instance):

```python
from horizon import workflows, forms

class SetInstanceDetailsAction(workflows.Action):
    availability_zone = forms.ChoiceField(label=_("Availability Zone"))
    name = forms.CharField(max_length=255, label=_("Instance Name"))
    flavor = forms.ChoiceField(label=_("Flavor"))
    count = forms.IntegerField(label=_("Number of Instances"), min_value=1)

    class Meta:
        name = _("Details")
        help_text = _("Describe the instance to be launched.")

class SetInstanceDetails(workflows.Step):
    action_class = SetInstanceDetailsAction
    contributes = ("availability_zone", "name", "flavor", "count")

class LaunchInstance(workflows.Workflow):
    slug = "launch_instance"
    name = _("Launch Instance")
    finalize_button_name = _("Launch")
    success_message = _('Launched %(count)s named "%(name)s".')
    failure_message = _('Unable to launch %(count)s named "%(name)s".')
    steps = (SetInstanceDetails, SetAccessControls, SetNetwork, PostCreationStep)

    def handle(self, request, context):
        try:
            api.nova.server_create(request, ...)
            return True
        except Exception:
            exceptions.handle(request)
            return False
```

## Plugin System for Third-Party Panels

Third-party OpenStack projects (Trove, Manila, Designate, Watcher, Ironic) ship Horizon plugins as separate Python packages. The pattern is:

1. Install the plugin package: `apt install python3-trove-dashboard` or `pip install trove-dashboard`
2. Copy or symlink the plugin's `enabled/` files into `/etc/openstack-dashboard/enabled/`
3. Run `manage.py compress && manage.py collectstatic`
4. Restart Apache

Example with Heat dashboard:

```bash
pip install heat-dashboard

# Copy enabled files
cp /usr/local/lib/python3.11/dist-packages/heat_dashboard/enabled/*.py \
   /etc/openstack-dashboard/enabled/

python3 /usr/share/openstack-dashboard/manage.py compress
python3 /usr/share/openstack-dashboard/manage.py collectstatic --noinput
systemctl restart apache2
```

## Angular (Legacy) vs Django Templates

Horizon has historically contained some panels built with AngularJS (the "Angular-based" workflow for Launch Instance, introduced in Liberty). These Angular panels are now considered legacy; new development uses Django class-based views with the DataTable/Form/Workflow framework.

The Angular panels are identifiable by their use of `<hz-magic-search>` and Angular directives in templates. They share state via `novaAPI`, `neutronAPI`, etc. AngularJS services defined in `openstack_dashboard/static/app/`.

New panels should use the pure Django/Python approach, which is better maintained and easier to customize.

## Horizon Settings Aggregation

At startup, Django loads settings from multiple sources in order:

1. `horizon/defaults.py` — Horizon framework defaults
2. `openstack_dashboard/defaults.py` — Dashboard application defaults
3. `openstack_dashboard/settings.py` — Merges everything; imports `local_settings.py`
4. `/etc/openstack-dashboard/local_settings.py` — Operator overrides
5. `enabled/_*.py` files — Plugin additions (merged into `INSTALLED_APPS`, `MIDDLEWARE`, etc.)

The `enabled/` files can use these keys to extend the Django configuration:
- `ADD_INSTALLED_APPS`: append to `INSTALLED_APPS`
- `ADD_MIDDLEWARE`: append to `MIDDLEWARE`
- `PANEL`: slug of a new panel to add
- `PANEL_GROUP`: slug of a panel group to add it to
- `PANEL_DASHBOARD`: slug of the dashboard to add it to
- `ADD_ANGULAR_MODULES`: list of AngularJS module names to register
- `ADD_JS_FILES`: additional JavaScript files to include in the bundle
- `ADD_SCSS_FILES`: additional SCSS files to compile into the theme
