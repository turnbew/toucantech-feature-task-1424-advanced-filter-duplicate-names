TASK's DATE: 06.04.2018 - 11.04.2018 

TASK's LEVEL: Medium (up level)

TASK SHORT DESCRIPTION: 1424 [
								"Add 'Duplicate name' filter in advanced filters > data cleaning.
								Use same options given in 'Missing name' filter in 'Data cleaning'.
								This filter will select users with the same selected 'Name' field.
								E.g if I build a filter of 'First name' and 'Last_name', it will select users with similar 
								first and last names (e.g if there were two users with the name: 'Ted Mosley', 
								it would select both of these users in the filter example above)."
							]


							GITHUB REPOSITORY CODE: feature/task-1424-advanced-filter-duplicate-names

ORIGINAL WORK: https://github.com/BusinessBecause/network-site/tree/feature/task-1424-advanced-filter-duplicate-names

CHANGES
 
	IN FILES: 
	
		\network-site\addons\default\modules\network_settings\controllers\members.php
	
			ADDED CODE: inside function runquery
				....
				$duplicateNameCounter = 0;
				....
				elseif($filter['id'] == 'duplicate_name') {
					array_push($subquery, array('type' => 'filter', 'value' => $filter));
					$duplicateNameCounter++;
				}
				.....
				if ($duplicateNameCounter > 1) $this->_adjustFiltersForDuplicateNames($query_parts);
	
			ADDED NEW FUNCTION 
			
				/** Set duplicate name filters on given mulitarray
				 * @input: $queryParts : multi-array : filter dields
				 * @return: void : working on $queryParts array
				**/
				private function _adjustFiltersForDuplicateNames(&$queryParts) 
				{	
					foreach ($queryParts as $key => &$subQueryPart) 
					{
						$filterDetails = $subQueryPart['details']; //array which contains the filter fields and operators

						#print_r($filterDetails); die; //for test			
						$mergeFilterFields = array();
						$cutKeys = array();
						
						#get duplicate_name filters
						$duplicateNameKeys = array();
						foreach ($filterDetails as $key => $details) 
							if ( isset($details['value']['id']) and $details['value']['id'] == 'duplicate_name') $duplicateNameKeys[] = $key;
						$keysCount = count($duplicateNameKeys);

						#print_r($duplicateNameKeys); die;	//for test			
						#if there are at least 2 duplicate_name keys in that case (if the OPERATOR between them is AND) they need to be merged
						if ($keysCount > 1) {
							$isSeparator = false;
							for ($i = 1; $i < $keysCount; $i++) {
								$duplicateNameKeyRecent = $duplicateNameKeys[$i]; 
								$duplicateNameKeyPrev = $duplicateNameKeys[$i-1];
								$operatorKeyPrev = $duplicateNameKeys[$i]-1;
								
								#we will check (double) that everything is OK to cut and merge filters
								$fieldRecent = $filterDetails[$duplicateNameKeyRecent]['value']['id'];
								$fieldPrev = $filterDetails[$duplicateNameKeyPrev]['value']['id'];
								$operatorPrev = (is_array($filterDetails[$operatorKeyPrev]['value'])) ? "" : strtolower($filterDetails[$operatorKeyPrev]['value']);			
								if ($fieldRecent == 'duplicate_name' and $operatorPrev == 'and' and $fieldPrev == 'duplicate_name') {
									$mergeFilterFields[] = $filterDetails[$duplicateNameKeyRecent]['value']['values']['value'];
									$mergeFilterFields[] = $filterDetails[$duplicateNameKeyPrev]['value']['values']['value'];
									$cutKeys[] = $duplicateNameKeyPrev;
									$cutKeys[] = $operatorKeyPrev;
									$cutKeys[] = $duplicateNameKeyRecent;
									$isSeparator = false;
								}
								else {
									if (count($cutKeys) > 0) {
										#workout and change $filterDetails
										$newField = 'CONCAT(' . implode(",", $mergeFilterFields) . ")";
										
										#cutKeys from $filterDetails array - expect first one - we put the new field there
										$filterDetails[$cutKeys[0]]['value']['values']['value'] = $newField;
										unset($filterDetails[$cutKeys[1]]); 
										unset($filterDetails[$cutKeys[2]]);
										
										#a small initialization
										$mergeFilterFields = array();
										$cutKeys = array();
										$isSeparator = true;
									}
								}
							}//END for loop
							
							
							if ( ! $isSeparator and count($cutKeys) > 0) {
								#workout and change $filterDetails
								$newField = 'CONCAT(' . implode(",", $mergeFilterFields) . ")";					
								#cutKeys from $filterDetails array - expect first one - we put the new field there
								$filterDetails[$cutKeys[0]]['value']['values']['value'] = $newField;
								unset($filterDetails[$cutKeys[1]]); 
								unset($filterDetails[$cutKeys[2]]);
							}					
						}//END if ($keysCount > 1)

						$subQueryPart['details'] = $filterDetails;
					}//END foreach loop
					#print_r($queryParts); die; //for test				
				} //END function _adjustFiltersForDuplicateNames
		
		
		
		
		\network-site\addons\default\modules\bbusers\models\user_query_m.php
		
			ADDED CODE: inside function _subquery_where_cond
			
				//duplicate names
				if ($field_slug == 'duplicate_name') {
					$field = '';
					
					switch (trim($field_values['value'])) {
						case 'title'	: $field = 'salutation'; 			break; 
						case 'first'	: $field = 'first_name'; 			break; 
						case 'preferred': $field = 'preferred_first_name'; 	break; 
						case 'middle'	: $field = 'middle_name'; 			break; 
						case 'last'		: $field = 'last_name'; 				break; 
						case 'whenat'	: $field = 'previous_last_name'; 	break; 
						#default - multiple field
						default 		: 		
							if (strpos($field_values['value'], 'CONCAT(') !== false and $details['type'] != 'filter') {
								$field = $field_values['value'];
								
								$from = array('title', 'first', 'preferred', 'middle', 'last', 'whenat');
								$to = array('salutation', 'first_name', 'preferred_first_name', 'middle_name', 'last_name', 'previous_last_name');
								
								$field = str_replace($from, $to, $field);
							}
							break; 
					}	

					if ($field != '') {
						$cond .= '( ' . $this->users_table.'.id IN (SELECT user_id FROM default_profiles WHERE '  . $field . ' IN ' . 
								 '	( SELECT ' . $field . ' FROM default_profiles ' . 
								 '		WHERE ' . $field . ' IS NOT NULL AND  ' . $field . ' <> "" ' .  
								 '		GROUP BY ' . $field . ' HAVING COUNT(*) > 1 ' . 
								 ')))';
					}
				} //End duplicate names filter
			
	
	
		\network-site\addons\default\modules\network_settings\views\members\partials\query_builder.php
		
			ADDED CODE: 
			
				<?=$this->load->view('members/partials/query_builder/duplicate_names_filter', array('value' => ''))?>
				
				
				
		\network-site\addons\default\modules\network_settings\views\members\partials\query_builder_filter.php
		
			ADDED CODE: 
			
				'duplicate_name' => 'duplicate_names_filter',
				
		
		\network-site\addons\default\modules\network_settings\views\reporting\partials\report_filter_add.php
		
			ADDED CODE: 
			
				'duplicate_name' =>array('duplicate_names_filter','query-colour-8-light'),
