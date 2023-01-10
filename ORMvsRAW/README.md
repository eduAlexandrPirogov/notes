# ORM vs SQL RAW

# Exapmle 1

Old:
```php
DB::table('working_lists')->select(['timezone', 'working_lists.id'])
            ->join('assign_list', 'working_lists.assign_id', '=', 'assign_list.id')
            ->join('enterprise_vehicle', 'enterprise_vehicle.vehicle_id', '=', 'assign_list.vehicle_id')
            ->join('enterprises', 'enterprises.id', '=', 'enterprise_vehicle.enterprise_id')
            ->whereIn('working_lists.id', $workingId)
            ->where('enterprise_vehicle.vehicle_id', '=', $vehicle)->get()->toArray();
```
Time executed: 3.82 s

New:
```php

        $enterprises = \DB::select("select * from working_lists wl 
        join assign_list al on al.id = wl.assign_id 
        join enterprise_vehicle ev on ev.vehicle_id = al.vehicle_id
        join enterprises e on e.id = ev.enterprise_id
        where wl.id in ('".$workingId."') and ev.vehicle_id = ".$vehicle);
```
Time executed: 1.8 s

# Example 2

Old:
```php
$id = Auth::guard('api')->user()->id;
        $withVehicles = \DB::select("select v.* from managers m 
        join enterprise_manager em on em.manager_id = m.id
        join enterprise_vehicle ev on ev.enterprise_id = ev.enterprise_id
        join vehicles v on v.id = ev.vehicle_id
        where m.id = $id");
        dd($withVehicles);
       // $withVehicles = Manager::with('vehicle')->where('id', '=', Auth::guard('api')->user()->id)->get();      
        return new ManagerResource($withVehicles);
```
Time executed:2.4s

New:
```php
$id = Auth::guard('api')->user()->id;
        $withVehicles = \DB::select("select v.* from managers m 
        join enterprise_manager em on em.manager_id = m.id
        join enterprise_vehicle ev on ev.enterprise_id = ev.enterprise_id
        join vehicles v on v.id = ev.vehicle_id
        where m.id = $id");
        dd($withVehicles);
       // $withVehicles = Manager::with('vehicle')->where('id', '=', Auth::guard('api')->user()->id)->get();      
        return new ManagerResource($withVehicles);
```
Time executed: 1.7s
