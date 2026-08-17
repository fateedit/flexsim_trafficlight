
# 红绿灯 blender 模型

> 红绿灯 3D 模型的 Blender 生成脚本：一键生成横臂式红绿灯（垂直立杆 + 水平横臂 + 红黄绿三色灯箱，含遮阳罩与底座）。
> 用途：生成的红绿灯模型导入 FlexSim，作为 1/2/3 号路口红绿灯实体的视觉外观，与占用模块及各路口配时方案配合使用。

import bpy
import math

# --- 清理场景 ---
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete(use_global=False)

# ==================== 参数 ====================
# 主立杆（垂直杆）
POLE_HEIGHT = 5.5          # 立杆总高度
POLE_RADIUS = 0.12         # 立杆半径

# 横臂（水平伸出的杆，沿X轴伸出到路面上方）
ARM_LENGTH = 2.5           # 横臂长度
ARM_RADIUS = 0.08          # 横臂半径
ARM_HEIGHT = 4.8           # 横臂离地高度

# 支撑斜杆
BRACE_RADIUS = 0.04        # 斜撑半径

# 横置灯箱（长方体面板）
PANEL_WIDTH = 1.8          # 灯箱长度（水平方向）
PANEL_HEIGHT = 0.5         # 灯箱高度（垂直方向）
PANEL_THICKNESS = 0.2      # 灯箱厚度

# 灯孔
HOLE_RADIUS = 0.18         # 灯孔半径
HOLE_DEPTH = 0.06          # 孔的深度
HOLE_SPACING = 0.55        # 灯孔圆心间距（水平方向）

# 遮阳罩（半圆管，在Y轴方向伸出）
VISOR_RADIUS = 0.21        # 遮阳罩内径
VISOR_LENGTH = 0.3         # 遮阳罩伸出长度（沿Y轴）
VISOR_THICKNESS = 0.03

# 面板从横臂向下偏移距离
PANEL_OFFSET_DOWN = 0.4

# 颜色 (R,G,B,A)
COLOR_POLE = (0.15, 0.15, 0.15, 1.0)
COLOR_PANEL = (0.1, 0.1, 0.1, 1.0)
COLOR_VISOR = (0.08, 0.08, 0.08, 1.0)
COLOR_ARM = (0.18, 0.18, 0.18, 1.0)
# =============================================

def create_material(name, color, emission=0.0):
    mat = bpy.data.materials.new(name=name)
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes["Principled BSDF"]
    bsdf.inputs['Base Color'].default_value = color[:3]
    bsdf.inputs['Roughness'].default_value = 0.6
    if emission > 0:
        bsdf.inputs['Emission Strength'].default_value = emission
        bsdf.inputs['Emission Color'].default_value = color[:3]
    return mat

# ==================== 1. 主立杆（垂直，Z轴） ====================
bpy.ops.mesh.primitive_cylinder_add(
    vertices=32, radius=POLE_RADIUS, depth=POLE_HEIGHT,
    location=(0, 0, POLE_HEIGHT / 2)
)
pole = bpy.context.object
pole.name = "Pole_Vertical"

# ==================== 2. 横臂（水平，沿X轴伸出） ====================
arm_x = ARM_LENGTH / 2
arm_y = 0
arm_z = ARM_HEIGHT

bpy.ops.mesh.primitive_cylinder_add(
    vertices=32, radius=ARM_RADIUS, depth=ARM_LENGTH,
    location=(arm_x, arm_y, arm_z),
    rotation=(0, math.radians(90), 0)  # 绕Y轴旋转90度，圆柱沿X轴
)
arm = bpy.context.object
arm.name = "Arm_Horizontal"

# ==================== 3. 连接节点 ====================
bpy.ops.mesh.primitive_cylinder_add(
    vertices=32, radius=POLE_RADIUS * 1.3, depth=POLE_RADIUS * 3,
    location=(0, 0, ARM_HEIGHT),
    rotation=(0, math.radians(90), 0)
)
joint = bpy.context.object
joint.name = "Joint"

# ==================== 4. 支撑斜杆 ====================
brace_start = (0, 0, ARM_HEIGHT - 1.2)
brace_end = (ARM_LENGTH * 0.6, 0, ARM_HEIGHT)

dx = brace_end[0] - brace_start[0]
dy = brace_end[1] - brace_start[1]
dz = brace_end[2] - brace_start[2]
brace_len = math.sqrt(dx**2 + dy**2 + dz**2)
brace_mid = (
    (brace_start[0] + brace_end[0]) / 2,
    (brace_start[1] + brace_end[1]) / 2,
    (brace_start[2] + brace_end[2]) / 2
)

angle_y = math.atan2(dx, dz)

bpy.ops.mesh.primitive_cylinder_add(
    vertices=16, radius=BRACE_RADIUS, depth=brace_len,
    location=brace_mid
)
brace = bpy.context.object
brace.name = "Brace"
brace.rotation_euler = (0, angle_y, 0)

# ==================== 5. 短竖杆（连接横臂末端和灯箱） ====================
PANEL_CENTER_X = ARM_LENGTH
PANEL_CENTER_Z = ARM_HEIGHT - PANEL_OFFSET_DOWN

bpy.ops.mesh.primitive_cylinder_add(
    vertices=32, radius=ARM_RADIUS * 0.9, depth=PANEL_OFFSET_DOWN,
    location=(PANEL_CENTER_X, 0, ARM_HEIGHT - PANEL_OFFSET_DOWN / 2)
)
hanger = bpy.context.object
hanger.name = "Hanger"

# ==================== 6. 横置灯箱（长方体） ====================
bpy.ops.mesh.primitive_cube_add(
    size=1.0,
    location=(PANEL_CENTER_X, PANEL_THICKNESS / 2, PANEL_CENTER_Z)
)
panel = bpy.context.object
panel.name = "Panel"
panel.scale = (PANEL_WIDTH / 2, PANEL_THICKNESS / 2, PANEL_HEIGHT / 2)
bpy.ops.object.transform_apply(scale=True)

# ==================== 7. 挖三个灯孔（水平排列在X轴方向） ====================
hole_positions = {
    'Red':    PANEL_CENTER_X - HOLE_SPACING,  # 左侧
    'Yellow': PANEL_CENTER_X,                  # 中间
    'Green':  PANEL_CENTER_X + HOLE_SPACING   # 右侧
}

# 挖孔（沿Y轴方向挖，因为面板面向Y轴）
for color_name, x_pos in hole_positions.items():
    bpy.ops.mesh.primitive_cylinder_add(
        vertices=32, radius=HOLE_RADIUS, depth=HOLE_DEPTH * 2,
        location=(x_pos, PANEL_THICKNESS + HOLE_DEPTH, PANEL_CENTER_Z),
        rotation=(math.radians(90), 0, 0)  # 沿Y轴方向
    )
    cutter = bpy.context.object
    cutter.name = f"Cutter_{color_name}"
    
    mod = panel.modifiers.new(name=f"Bool_{color_name}", type='BOOLEAN')
    mod.operation = 'DIFFERENCE'
    mod.object = cutter
    
    bpy.context.view_layer.objects.active = panel
    bpy.ops.object.modifier_apply(modifier=mod.name)
    bpy.data.objects.remove(cutter, do_unlink=True)

# ==================== 8. 三个灯光圆片 ====================
light_colors = {
    'Red':    (1.0, 0.1, 0.1, 1.0),
    'Yellow': (1.0, 0.85, 0.0, 1.0),
    'Green':  (0.1, 0.9, 0.2, 1.0)
}
light_objects = {}

for color_name, x_pos in hole_positions.items():
    bpy.ops.mesh.primitive_cylinder_add(
        vertices=32, radius=HOLE_RADIUS * 0.9, depth=HOLE_DEPTH * 0.6,
        location=(x_pos, PANEL_THICKNESS - HOLE_DEPTH * 0.2, PANEL_CENTER_Z),
        rotation=(math.radians(90), 0, 0)  # 沿Y轴方向
    )
    light = bpy.context.object
    light.name = f"Light_{color_name}"
    light_objects[color_name] = light

# ==================== 9. 遮阳罩（半圆管，沿Y轴伸出，在XZ平面形成半圆） ====================
for color_name, x_pos in hole_positions.items():
    # 创建完整圆管（沿Y轴方向，从面板向前伸出）
    bpy.ops.mesh.primitive_cylinder_add(
        vertices=64, radius=VISOR_RADIUS, depth=VISOR_LENGTH,
        location=(x_pos, PANEL_THICKNESS + VISOR_LENGTH / 2, PANEL_CENTER_Z),
        rotation=(math.radians(90), 0, 0)  # 绕X轴旋转90度，圆柱沿Y轴
    )
    visor_outer = bpy.context.object
    visor_outer.name = f"VisorOuter_{color_name}"
    
    # 挖空内部
    bpy.ops.mesh.primitive_cylinder_add(
        vertices=32, radius=VISOR_RADIUS - VISOR_THICKNESS, depth=VISOR_LENGTH + 0.02,
        location=(x_pos, PANEL_THICKNESS + VISOR_LENGTH / 2, PANEL_CENTER_Z),
        rotation=(math.radians(90), 0, 0)  # 沿Y轴方向
    )
    inner = bpy.context.object
    inner.name = f"Inner_{color_name}"
    
    mod = visor_outer.modifiers.new(name="Bool_Hollow", type='BOOLEAN')
    mod.operation = 'DIFFERENCE'
    mod.object = inner
    bpy.context.view_layer.objects.active = visor_outer
    bpy.ops.object.modifier_apply(modifier=mod.name)
    bpy.data.objects.remove(inner, do_unlink=True)
    
    # 切掉下半部分（在Z轴方向的下半部分），保留上半部分作为遮阳罩
    bpy.ops.mesh.primitive_cube_add(
        size=VISOR_RADIUS * 3,
        location=(x_pos, PANEL_THICKNESS + VISOR_LENGTH / 2, PANEL_CENTER_Z - VISOR_RADIUS)
    )
    cutter_half = bpy.context.object
    cutter_half.name = f"HalfCutter_{color_name}"
    
    mod = visor_outer.modifiers.new(name="Bool_Half", type='BOOLEAN')
    mod.operation = 'DIFFERENCE'
    mod.object = cutter_half
    bpy.context.view_layer.objects.active = visor_outer
    bpy.ops.object.modifier_apply(modifier=mod.name)
    bpy.data.objects.remove(cutter_half, do_unlink=True)
    
    visor_outer.name = f"Visor_{color_name}"

# ==================== 10. 底座 ====================
bpy.ops.mesh.primitive_cylinder_add(
    vertices=32, radius=0.25, depth=0.2, location=(0, 0, 0.1)
)
base = bpy.context.object
base.name = "Base"

bpy.ops.mesh.primitive_cylinder_add(
    vertices=6, radius=0.35, depth=0.05, location=(0, 0, 0.025)
)
flange = bpy.context.object
flange.name = "Flange"

# ==================== 11. 横臂末端装饰帽 ====================
bpy.ops.mesh.primitive_cylinder_add(
    vertices=32, radius=ARM_RADIUS * 1.4, depth=ARM_RADIUS * 2,
    location=(ARM_LENGTH, 0, ARM_HEIGHT),
    rotation=(0, math.radians(90), 0)
)
end_cap = bpy.context.object
end_cap.name = "EndCap"

# ==================== 12. 材质 ====================
mat_pole = create_material("M_Pole", COLOR_POLE)
mat_arm = create_material("M_Arm", COLOR_ARM)
mat_panel = create_material("M_Panel", COLOR_PANEL)
mat_visor = create_material("M_Visor", COLOR_VISOR)

for obj in [pole, base, flange, joint]:
    obj.data.materials.append(mat_pole)

for obj in [arm, brace, hanger, end_cap]:
    obj.data.materials.append(mat_arm)

panel.data.materials.append(mat_panel)

# 灯光材质（带自发光）
mat_red = create_material("M_Red", light_colors['Red'], emission=8.0)
mat_yellow = create_material("M_Yellow", light_colors['Yellow'], emission=8.0)
mat_green = create_material("M_Green", light_colors['Green'], emission=8.0)

light_objects['Red'].data.materials.append(mat_red)
light_objects['Yellow'].data.materials.append(mat_yellow)
light_objects['Green'].data.materials.append(mat_green)

for name in hole_positions:
    visor = bpy.data.objects.get(f"Visor_{name}")
    if visor:
        visor.data.materials.append(mat_visor)

# ==================== 13. 打包到空物体 ====================
bpy.ops.object.select_all(action='DESELECT')
all_objs = [pole, arm, joint, brace, hanger, panel, base, flange, end_cap]
all_objs.extend(list(light_objects.values()))

for name in hole_positions:
    v = bpy.data.objects.get(f"Visor_{name}")
    if v:
        all_objs.append(v)

for obj in all_objs:
    obj.select_set(True)

bpy.ops.object.empty_add(type='PLAIN_AXES', location=(0, 0, 0))
rig = bpy.context.object
rig.name = "TrafficLight_Arm"

for obj in all_objs:
    obj.parent = rig

print("✅ 横臂式横向红绿灯模型生成完毕！")
print("结构：垂直立杆 + 水平横臂(X轴) + 横向灯箱(红黄绿水平排列)")
print(f"横臂沿X轴伸出 {ARM_LENGTH}m，高度 {ARM_HEIGHT}m")
print("灯箱横向排列：🔴红色(左) 🟡黄色(中) 🟢绿色(右)")
print("遮阳罩：半圆管沿Y轴伸出，在XZ平面形成半圆遮阳")
print("选中 'TrafficLight_Arm' 空物体即可整体移动。")

---

## Related

- [占用模块](占用模块.md) — 红绿灯占用的控制逻辑脚本，视觉模型需配合占用模块共同工作
- 收费站blender模型 — 收费站的 3D 视觉模型，同为场景可视化组件
- [1号路口](1号路口-两相位方案.md) — 1 号路口的完整配置，blender 模型的实际应用场景
