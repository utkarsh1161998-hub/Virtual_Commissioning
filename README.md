import asyncio
from asyncua import Client
import sys

async def main():
    url = "opc.tcp://127.0.0.1:4840/freeopcua/server/"
    print(f"Connecting Digital Twin to virtual PLC at {url}...")
    
    try:
        async with Client(url=url) as client:
            # Locate the PLC tags via their Node IDs
            # (In a real setup, you would browse or use standard OPC tags)
            ns_idx = 2
            motor_tag = await client.nodes.root.get_child(["0:Objects", f"{ns_idx}:PLC_Tags", f"{ns_idx}:Motor_Run"])
            sensor_tag = await client.nodes.root.get_child(["0:Objects", f"{ns_idx}:PLC_Tags", f"{ns_idx}:Sensor_PartDetected"])
            
            # Physical parameters of our virtual conveyor belt
            package_position = 0.0  # Meters along the belt
            conveyor_length = 1.0   # Meters
            sensor_position = 0.9   # Sensor sits near the end
            speed = 0.1             # Moves at 0.1 meters per second
            dt = 0.1                # Time step (100ms)
            
            print("\nDigital Twin Active. Simulating Physics...")
            print("--------------------------------------------------")
            
            while True:
                # 1. Read Actuator Commands from the PLC
                motor_on = await motor_tag.get_value()
                
                # 2. Simulate Physical Dynamics
                if motor_on:
                    package_position += speed * dt
                
                # 3. Evaluate Virtual Sensors
                if package_position >= sensor_position:
                    part_detected = True
                else:
                    part_detected = False
                    
                # 4. Write Sensor State back to PLC
                await sensor_tag.write_value(part_detected)
                
                # Print terminal visualization of the physical state
                belt_visual = ["_"] * 10
                idx = min(int(package_position * 10), 9)
                belt_visual[idx] = "📦"
                sensor_marker = "🚨" if part_detected else "⚪"
                
                sys.stdout.write(f"\r[{''.join(belt_visual)}] Pos: {package_position:.2f}m | Sensor: {sensor_marker} | Motor: {'▶️' if motor_on else '⏹️'}")
                sys.stdout.flush()
                
                await asyncio.sleep(dt)
                
    except Exception as e:
        print(f"\nConnection failed or lost: {e}")

if __name__ == "__main__":
    asyncio.run(main())
